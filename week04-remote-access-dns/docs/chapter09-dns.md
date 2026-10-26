# Chapter 9 — DNS 원리 및 BIND 서버 구축

> **핵심 질문:** `web.lab.test`라는 이름을 `192.168.199.132`로 변환한 것은 누가, 어떤 데이터로, 어떤 네트워크 절차를 통해 수행했을까?

[메인 README로 돌아가기](../README.md)

## 1. DNS가 필요한 이유

사람은 `web.lab.test` 같은 도메인 이름을 기억하기 쉽지만, 실제 네트워크 통신은 IP 주소를 기반으로 한다. DNS(Domain Name System)는 **도메인 이름과 여러 종류의 DNS 레코드를 분산·계층적으로 관리하고 질의에 응답하는 시스템**이다.

**주의:** DNS는 단순히 이름과 IP를 1:1로 변환하는 표에 한정되지 않는다. 메일 서버(MX), 서비스 정책(TXT), 위임(NS) 등 다양한 정보를 저장한다.

### 핵심 구성 요소

| 용어                      | 역할                                    | 이번 실습에서                                  |
| ------------------------- | --------------------------------------- | ---------------------------------------------- |
| Stub resolver/클라이언트  | OS·프로그램에서 DNS 질의 수행           | Windows `Resolve-DnsName`, Linux `dig`         |
| Recursive Resolver        | 대신 질의를 수행하고 캐시해 응답        | 이번 BIND 서버의 주 기능 아님                  |
| Authoritative Name Server | Zone의 DNS 레코드에 대해 권한 있는 응답 | Rocky `named`가 `lab.test` 및 역방향 Zone 제공 |
| Zone                      | 특정 DNS 영역의 데이터·권한 관리 단위   | `lab.test`, `199.168.192.in-addr.arpa`         |
| Resource Record           | DNS 데이터 한 건                        | A, NS, SOA, PTR                                |

## 2. 인터넷 DNS는 어떻게 이름을 찾을까?

캐시와 별도 구성에 따른 예외를 제외하면, 일반적인 인터넷 이름 해석은 다음 과정으로 이해할 수 있다.

```text
브라우저/운영체제
  ↓ (로컬 캐시·hosts 파일 등을 먼저 참조할 수 있음)
재귀 DNS 리졸버 (없으면 외부 서버에 질의)
  ↓ 루트(.) 서버에 안내 요청
Root DNS → .com 네임 서버 정보
  ↓
TLD(.com) DNS → example.com 권한 서버 정보
  ↓
Authoritative DNS → www.example.com에 대한 실제 레코드
  ↓
Recursive Resolver가 응답을 캐시하고 클라이언트에 전달
```

- **재귀 질의**: 클라이언트는 재귀 서버에 최종 답을 구해 달라고 요청한다.
- **반복 질의**: 리졸버가 다른 서버로부터 '다음 서버를 찾아가라'는 위임 정보를 받고 계속 조회할 수 있다.
- **권한 서버**는 자신이 관리하는 Zone의 레코드를 제공한다.
- **이번 실습**에서는 Windows가 Rocky 서버 IP를 **직접 지정**해 `lab.test`에 대한 답을 받았으므로 인터넷 루트→TLD 조회를 수행할 필요가 없었다.

## 3. 주요 DNS 레코드

| 타입  | 의미                              | 예시                                                   |
| ----- | --------------------------------- | ------------------------------------------------------ |
| A     | 호스트 이름 → IPv4                | `web.lab.test. IN A 192.168.199.132`                   |
| AAAA  | 호스트 이름 → IPv6                | `host.example. IN AAAA 2001:db8::1` (문서용 예시 주소) |
| PTR   | 역방향 조회(IP → 이름)            | `132 IN PTR web.lab.test.`                             |
| NS    | 특정 Zone을 담당하는 서버         | `lab.test. IN NS ns.lab.test.`                         |
| SOA   | Zone 관리 책임·Serial·타이머 정보 | Zone 처음에 작성                                       |
| CNAME | 별칭을 다른 **이름**에 연결       | `alias IN CNAME web.lab.test.`                         |
| MX    | 도메인 메일 처리 서버 지정        | `@ IN MX 10 mail.example.`                             |
| TXT   | 임의 텍스트 데이터                | SPF 검증 등에서 사용 가능                              |

**자주 하는 혼동:** PTR이 있다고 A가 자동 생성되지는 않는다. 정방향과 역방향은 각각의 Zone과 레코드를 따로 설정한다. PTR 결과는 신원 인증서나 신뢰 가능한 사용자 증명을 대신하지 않는다.

### TTL(Time To Live)

- 레코드의 TTL이 3600초라면 DNS 캐시는 일반적으로 최대 1시간 해당 응답을 재사용할 수 있다.
- TTL이 짧으면 레코드 변경을 더 빨리 반영하기 유리하지만 조회 트래픽이 증가할 수 있다.
- TTL이 길면 질의 부하를 줄일 수 있지만 변경된 주소가 클라이언트에 반영되기까지 오래 걸릴 수 있다.
- Windows `Resolve-DnsName`에서 확인한 실제 A 레코드의 TTL은 **3600**이었다.

## 4. BIND 기본 구조

Rocky Linux에서 BIND 패키지와 도구를 설치해 DNS 서버를 구축했다.

```bash
sudo dnf install -y bind bind-utils
rpm -q bind bind-utils
sudo cp -a /etc/named.conf /etc/named.conf.bak-week04
```

### 구성 파일 역할

| 경로                         | 무엇을 정의?                                     |
| ---------------------------- | ------------------------------------------------ |
| `/etc/named.conf`            | `listen-on`, 재귀 질의, ACL, Zone 정의 등의 설정 |
| `/var/named/lab.test.zone`   | 정방향 SOA·NS·A 레코드                           |
| `/var/named/199.168.192.rev` | 역방향 SOA·NS·PTR 레코드                         |
| `/etc/named.conf.bak-week04` | 설정 변경 전 백업                                |

BIND가 기본 설정에 따라 Zone 파일 경로를 해석하므로, `file "lab.test.zone";`처럼 상대경로를 사용하는 경우 `options` 블록의 `directory` 설정을 확인해야 한다.

### 실습에서 사용한 BIND 설정의 핵심 뜻

- `listen-on port 53 { 127.0.0.1; 192.168.199.132; };`: 해당 IPv4 인터페이스에서 DNS 요청을 받는다.
- `zone "lab.test"`: 관리할 정방향 영역 지정.
- `type master`: 실습의 기본 권한 Zone으로 사용(현대 BIND 문법에서는 primary도 사용).
- `allow-query { localhost; 192.168.199.0/24; };`: 허용된 클라이언트에서 해당 Zone에 질의 가능하도록 제한.
- `allow-update { none; };`: 동적 DNS 갱신 거부.
- `allow-transfer { none; };`: 해당 Zone의 전송 거부.
- `recursion no;`: 외부 도메인을 찾아주는 재귀 리졸버 역할 비활성화.

⚠️ 이 옵션은 **학습 및 권장 설정 해설**이다. 실제 화면에서는 `named.conf` 전체 내용이 공개되지 않았으므로 `recursion no;`, `allow-transfer { none; };` 등이 실제로 적용되었다고 단정하지 않는다. 아래 명령으로 별도 확인할 수 있다.

```bash
sudo grep -nE 'listen-on|recursion|allow-query|allow-transfer|allow-update' /etc/named.conf
sudo named-checkconf -z
```

## 5. 정방향 Zone의 내부 구조

예시: `/var/named/lab.test.zone`

```dns
$TTL 1H
@ IN SOA ns.lab.test. admin.lab.test. (
    2026102701 ; serial
    1H         ; refresh
    15M        ; retry
    1W         ; expire
    1H         ; negative cache TTL field
)

@   IN NS ns.lab.test.
ns  IN A 192.168.199.132
web IN A 192.168.199.132
```

이 예시는 실습 때 사용한 내용에 근거해 재작성한 **학습 참고 파일**이며 실제 서버의 현재 파일 덤프가 아니다.

| 항목              | 의미                                                                      |
| ----------------- | ------------------------------------------------------------------------- |
| `$TTL 1H`         | TTL을 별도로 지정하지 않은 RR의 기본 TTL 1시간                            |
| `@`               | 현재 Zone 기점(`lab.test.`)                                               |
| `SOA`             | 관리 서버, 관리자 연락처, Serial 및 갱신 시간 설정                        |
| `ns.lab.test.`    | Primary 이름 서버 이름                                                    |
| `admin.lab.test.` | 관리자 이메일 `admin@lab.test`에 대응하는 DNS 표기                        |
| `2026102701`      | Zone Serial. 내용을 바꿨을 때 증가시켜 secondary 서버 갱신 판단 등에 사용 |
| `IN`              | Internet Class                                                            |
| `NS`              | Zone의 권한 서버를 가리키는 레코드                                        |
| `ns`              | `ns.lab.test.`를 뜻하는 상대적 소유자 이름                                |
| `web`             | `web.lab.test.`를 뜻하는 상대적 소유자 이름                               |
| 이름 끝의 `.`     | 절대 FQDN. 생략하면 Zone 원점이 덧붙을 수 있음                            |

SOA의 마지막 1H는 현대 DNS에서 **부정 응답 캐시 TTL 설정에 관여하는 필드**다. 과거 문서에서 표시하는 'minimum TTL'과 혼동하지 말자.

### 정방향 확인

```bash
sudo named-checkconf -z
sudo named-checkzone lab.test /var/named/lab.test.zone
sudo systemctl enable --now named
sudo systemctl reload named

dig @127.0.0.1 web.lab.test A +short
dig @192.168.199.132 web.lab.test A +short
```

실제 확인한 결과:

```text
zone lab.test/IN: loaded serial 2026102701
OK
192.168.199.132
```

**Windows PowerShell:**

```powershell
Resolve-DnsName -Name web.lab.test -Type A -Server 192.168.199.132
```

실제 Windows 출력에서 `web.lab.test / A / TTL 3600 / Answer / 192.168.199.132` 확인.

## 6. 역방향 Zone과 PTR의 원리

정방향이 `이름 → IP`이면, 역방향은 `IP → 이름`이야.

```text
IP:            192.168.199.132
Reverse query: 132.199.168.192.in-addr.arpa
/24 zone:      199.168.192.in-addr.arpa
PTR owner:     132
PTR target:    web.lab.test.
```

예시: `/var/named/199.168.192.rev`

```dns
$TTL 1H
@ IN SOA ns.lab.test. admin.lab.test. (
    2026102701
    1H
    15M
    1W
    1H
)
@   IN NS ns.lab.test.
132 IN PTR web.lab.test.
```

`named.conf`에서 다음 Zone 선언이 필요하다.

```conf
zone "199.168.192.in-addr.arpa" IN {
    type master;
    file "199.168.192.rev";
    allow-query { localhost; 192.168.199.0/24; };
    allow-update { none; };
    allow-transfer { none; };
};
```

### 역방향 조회 검증

```bash
sudo named-checkzone 199.168.192.in-addr.arpa /var/named/199.168.192.rev
sudo systemctl reload named
dig @192.168.199.132 -x 192.168.199.132 +short
# web.lab.test.
```

**Windows PowerShell:**

```powershell
nslookup 192.168.199.132 192.168.199.132
```

실제 출력: 서버 이름 및 조회 이름으로 `web.lab.test`, 주소 `192.168.199.132`가 나타남.

- `nslookup`의 첫 번째 `서버:`는 **질의에 응답한 DNS 서버의 표시 이름**이며, 두 번째 이름은 **질의한 IP에 대한 PTR 답변**이다. 이번에는 둘이 같은 IP이므로 같은 이름으로 표시될 수 있다.
- Windows에서 `-Server 192.168.199.132` 또는 `nslookup ... 192.168.199.132`는 **지정 서버에 질의**하는 것이지 OS 전체 DNS 구성을 변경하는 게 아니다.

## 7. 포트·방화벽·네트워크 경계

### DNS가 UDP 53과 TCP 53을 모두 사용하는 이유

- 일반적인 짧은 질의와 응답은 UDP 53에서 널리 처리된다.
- 응답 크기, 정책, 패킷 잘림(TC), Zone 전송 등 여러 상황에서 TCP 53도 사용된다.
- 따라서 네임 서버 운영에서는 TCP/UDP 53을 모두 고려해야 한다.
- 한 줄로 'DNS는 UDP만 쓴다'라고 설명하면 부정확하다.

### `ss` 출력 해석

```bash
sudo ss -lntup | grep ':53'
```

실제 화면에서 `named`가 `127.0.0.1:53`, `192.168.199.132:53`에 바인딩되어 있었다. `avahi-daemon`의 `:5353`도 같이 나올 수 있으므로 **53번과 5353번은 다른 포트**임을 확인한다.

### firewalld

```bash
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --zone=public --add-service=dns
```

- 스크린샷에서 활성 존은 `public`, 인터페이스는 `ens160`이었다.
- `--add-service=dns`는 firewalld의 DNS 서비스 정의를 허용한다.
- 위 명령은 **runtime 정책**이므로 시스템 재시작 후 지속되지 않을 수 있다.
- 이번 실습은 **같은 VMware NAT 사설망에서 접속 성공을 확인**한 것이지 인터넷에 DNS를 공개했다는 의미가 아니다.
- 실제 운영에서는 방화벽 원본 주소 제한을 별도로 검토하고 ACL·수신 인터페이스·비재귀 설정을 함께 적용해야 한다.

## 8. DNS 장애 대응 체크리스트

### A. Rocky 내부에서 질의가 안 됨

```bash
systemctl status named
sudo named-checkconf -z
sudo named-checkzone lab.test /var/named/lab.test.zone
sudo journalctl -u named -b --no-pager -n 50
dig @127.0.0.1 web.lab.test A
```

우선 BIND 자체 문제인가, Zone 문법 오류인가, 레코드가 누락됐는가를 구분한다.

### B. Rocky에서는 되는데 Windows에서는 실패

```bash
ip -br addr
sudo ss -lntup
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --list-all --zone=public
sudo grep -nE 'listen-on|allow-query|recursion' /etc/named.conf
```

- `named`가 루프백에만 바인딩되어 있지 않은지?
- Windows가 실제로 VM의 IP로 접근할 수 있는지?
- 방화벽에서 DNS 서비스가 활성 존에 허용되어 있는지?
- `allow-query`에서 Windows/VMware NAT 대역을 허용했는지?
- Windows에서 `-Server` 인자에 **올바른 IP**를 지정했는지?

### C. `SERVFAIL`, `REFUSED`, `NXDOMAIN` 구분

| 응답/상태  | 의미를 추적할 방향                                             |
| ---------- | -------------------------------------------------------------- |
| `NOERROR`  | 질의 자체는 성공. 답변 RR이 없을 수도 있으므로 Answer 확인     |
| `NXDOMAIN` | 요청한 이름이 존재하지 않음                                    |
| `REFUSED`  | 서버 정책상 질의 거부(ACL 등 확인)                             |
| `SERVFAIL` | 서버가 질의를 처리하는 과정에서 실패함(로그·설정·위임 등 확인) |
| timeout    | 서버 연결, ACL/방화벽, IP, 네트워크, UDP/TCP 등 확인           |

### D. 레코드를 수정했는데 예전 값이 계속 나옴

- 설정 파일을 저장하고 `named-checkzone`을 다시 실행했는지?
- Zone Serial을 증가시켰는지?
- `systemctl reload named`로 적용했는지?
- 클라이언트/리졸버 캐시 TTL이 남아 있는지?
- 실제 응답을 주는 서버가 원하는 DNS 서버인지 `dig @SERVER`로 지정했는지?

## 9. 운영 보안 관점

- 권한 DNS 서버에 굳이 재귀 DNS 기능을 공개하지 않는다. 일반적인 권한 전용 서버는 `recursion no;`를 검토한다.
- `allow-query`를 필요한 내부 네트워크로 한정한다.
- 무단 Zone 전송(AXFR)을 차단하거나 승인된 secondary 서버만 허용한다.
- `allow-update { none; };` 등 동적 업데이트 정책을 명시한다.
- DNS 설정은 `named-checkconf`, `named-checkzone`으로 검사하고 로그를 남긴다.
- 서비스·포트·방화벽·ACL을 나눠 점검하며, 실습용 사설 IP를 운영 외부망에 그대로 배포하지 않는다.
- **DNSSEC**는 DNS 데이터 출처·무결성 검증에 도움을 주지만, 통신 자체를 암호화하는 TLS와 같은 기술은 아니다. 이번에는 별도 구현하지 않았다.

## 10. 간단 요약

> DNS는 도메인 이름을 IP 주소로 바꾸는 것에 그치지 않고, 계층적인 네임 서버와 각종 레코드로 이름 관련 정보를 제공하는 시스템이다. 이번에는 Rocky Linux에 BIND를 설치해서 `lab.test` 권한 Zone을 만들고, A 레코드로 `web.lab.test`를 서버 IP에 연결했다. 이어 `199.168.192.in-addr.arpa` Zone에 PTR을 추가하여 역방향 조회도 구성했다. `named-checkconf`, `named-checkzone`으로 설정을 검증하고, Linux뿐 아니라 Windows에서 DNS 서버를 지정해 질의했을 때 원하는 결과가 나오는지 확인했다.

## 11. 핵심 확인 질문

1. 왜 로컬 서버에 Zone이 있으면 Root/TLD를 조회할 필요가 없을까?
2. Authoritative 서버와 Recursive Resolver는 어떤 역할 차이가 있을까?
3. 정방향 A와 역방향 PTR은 왜 별도 레코드일까?
4. Zone 파일의 `@`, 마지막 점, `Serial`, `TTL`의 의미는?
5. DNS는 왜 TCP와 UDP를 모두 사용할까?
6. Windows가 지정 DNS 서버에 성공적으로 조회했다고 해서 기본 DNS가 변경된 것은 왜 아닐까?
7. DNS가 응답하지 않을 때 어떻게 `named` → Port → Firewall → ACL → Zone 순으로 확인할까?

[예상 질문 답안에서 확인](review-questions.md)

## 공식 참고자료

- [Rocky Linux BIND Private DNS Server](https://docs.rockylinux.org/10/guides/dns/private_dns_server_using_bind/)
- [BIND 9 Configuration Reference](https://bind9.readthedocs.io/en/latest/reference.html)
- [RFC 6761 — Special-Use Domain Names (`.test`)](https://www.rfc-editor.org/rfc/rfc6761)
- `man named.conf`, `man dig`, `man nslookup`, `man firewall-cmd`
