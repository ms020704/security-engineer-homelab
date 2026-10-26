# Week04 명령어 사전 — 무엇을/왜/어디에서/무엇을 확인?

> 문법을 외우는 데서 끝내지 않고, 각 명령으로 확인하려는 **운영상 가설**을 설명하기 위한 복습 문서. `PS>`는 Windows PowerShell, `$`는 Rocky Linux Bash를 의미한다.

[메인 README](../README.md) · [Chapter 8](chapter08-remote-access.md) · [Chapter 9](chapter09-dns.md)

## 1. SSH 서버 점검 — Rocky Linux

| 명령어                             | 기능과 옵션                                         | 왜 실행하나?                     | 기대 결과 / 해석             |
| ---------------------------------- | --------------------------------------------------- | -------------------------------- | ---------------------------- |
| `hostname -I`                      | 호스트의 IP 주소 목록                               | 접속할 IP 확인                   | `192.168.199.132`            |
| `systemctl is-active sshd`         | 서비스 현재 실행 상태                               | SSH 데몬이 동작하나?             | `active`                     |
| `systemctl is-enabled sshd`        | 자동 시작 설정                                      | 재부팅 뒤에도 서비스가 시작되나? | `enabled`                    |
| `sudo ss -tlnp`                    | TCP(`-t`), LISTEN(`-l`), 숫자(`-n`), 프로세스(`-p`) | 실제 포트가 열려 있나?           | `:22`의 sshd 프로세스        |
| `sudo sshd -T`                     | SSH 데몬의 적용 설정 출력                           | 인증 정책·리스닝 포트 확인       | `port`, `permitrootlogin` 등 |
| `sudo sshd -t`                     | SSH 구성 파일 문법 확인                             | 설정 변경 전 오류 예방           | 성공 시 출력 없음            |
| `sudo journalctl -u sshd -b -n 50` | sshd 로그, 현재 부팅, 최근 50개                     | 왜 접속/인증 실패했나?           | 실패·성공 이유 분석          |

**지식 확인:** `is-active`는 서비스 현재 상태, `is-enabled`는 부팅 시 활성화 설정이다. 둘은 달라질 수 있다.

## 2. 공개키 인증 및 SCP — Windows PowerShell

```powershell
ssh-keygen -t ed25519 -f "$env:USERPROFILE\.ssh\id_ed25519_rocky_lab" -C "rocky-lab"
```

- `-t ed25519`: SSH 키 알고리즘 지정
- `-f`: 키 파일 위치 지정(개인키와 `.pub` 파일 생성)
- `-C`: 공개키 끝에 붙는 설명 주석
- 개인키가 이미 존재하면 **덮어쓰지 말 것**

```powershell
Get-Content "$env:USERPROFILE\.ssh\id_ed25519_rocky_lab.pub" |
    ssh msk@192.168.199.132 "umask 077; mkdir -p ~/.ssh; cat >> ~/.ssh/authorized_keys"
```

- `Get-Content`: Windows 공개키 파일 읽기
- `|`: PowerShell의 출력을 ssh의 표준 입력으로 전달
- `umask 077`: 서버가 새로 생성하는 파일·디렉터리의 기본 권한을 제한
- `cat >> authorized_keys`: 허용된 **사용자 공개키**를 서버 계정 목록에 추가
- 주의: 이 명령은 두 번 실행하면 같은 키가 중복 등록될 수 있으므로 재현 시 중복 확인이 필요

```powershell
ssh -i "$env:USERPROFILE\.ssh\id_ed25519_rocky_lab" `
    -o PreferredAuthentications=publickey -o PasswordAuthentication=no `
    msk@192.168.199.132 "whoami; hostname"
```

- `-i`: 개인키 파일 지정
- `-o`: OpenSSH 옵션 지정
- `PreferredAuthentications=publickey`: 공개키 방식 우선
- `PasswordAuthentication=no`: **이 실행에서** 비밀번호 인증으로 대체하지 않음
- `"whoami; hostname"`: 원격 서버에서 두 명령 실행
- passphrase를 입력한 것은 **개인키 잠금 해제**이며 서버 계정 비밀번호 인증이 아님

```powershell
scp -i "$env:USERPROFILE\.ssh\id_ed25519_rocky_lab" `
    "$env:TEMP\week04.txt" "msk@192.168.199.132:/home/msk/week04.txt"
```

- `scp`: SSH 기반 파일 복사 명령
- `-i`: 개인키 지정
- `로컬파일 서버:원격파일`: Windows에서 Rocky로 전송
- Linux에서 `cat ~/week04.txt`로 파일 내용을 확인해야 완료를 증명할 수 있음

## 3. BIND 패키지·서비스 — Rocky Linux

| 명령어                                                  | 의미                                              | 확인 포인트                 |
| ------------------------------------------------------- | ------------------------------------------------- | --------------------------- |
| `sudo dnf install -y bind bind-utils`                   | BIND 서버+DNS 질의 도구 설치, `-y` 확인 자동 승인 | 설치 완료 여부              |
| `rpm -q bind bind-utils`                                | 현재 설치된 패키지 버전 조회                      | 설치 확인                   |
| `sudo cp -a /etc/named.conf /etc/named.conf.bak-week04` | 속성 보존하여 설정 백업                           | 변경 전 복구 지점           |
| `sudo nano /etc/named.conf`                             | BIND 서버 전체 설정 편집                          | options, zone 선언 위치     |
| `sudo nano /var/named/lab.test.zone`                    | 정방향 Zone 편집                                  | SOA, NS, A 레코드           |
| `sudo nano /var/named/199.168.192.rev`                  | 역방향 Zone 편집                                  | SOA, NS, PTR 레코드         |
| `sudo chown root:named 파일`                            | 파일 소유자·그룹 지정                             | `named`가 읽을 수 있도록    |
| `sudo chmod 640 파일`                                   | owner RW/group R/others 없음                      | 최소 권한                   |
| `sudo restorecon -v 파일`                               | SELinux 파일 컨텍스트를 정책에 맞게 복원          | 정책상 파일 접근 허용       |
| `sudo named-checkconf`                                  | named.conf 구문 검증                              | 성공 시 출력이 없을 수 있음 |
| `sudo named-checkconf -z`                               | 선언된 Zone 적재 검사까지 수행                    | `loaded serial` 확인        |
| `sudo named-checkzone 도메인 파일`                      | Zone 파일 유효성 검증                             | `OK`                        |
| `sudo systemctl enable --now named`                     | 부팅 연동 + 지금 실행                             | `enabled`, `active`         |
| `sudo systemctl reload named`                           | 서비스 설정 다시 읽기                             | 설정 변경 반영              |
| `sudo journalctl -u named -b -n 50`                     | named 로그 확인                                   | 구문·로드·권한 문제         |

**주의:** `named-checkconf`가 성공해도 DNS 질의가 성공한 것은 아니다. `named-checkzone`과 클라이언트 `dig`/`Resolve-DnsName`까지 분리 검증한다.

## 4. DNS 수신 및 방화벽 — Rocky Linux

```bash
sudo ss -lntup | grep ':53'
```

- `-l`: 리스닝 또는 수신 대기 소켓
- `-n`: 포트를 서비스명으로 바꾸지 않고 숫자로 표시
- `-t`: TCP / `-u`: UDP / `-p`: 프로세스
- 실제 화면에서 `named`의 `192.168.199.132:53` TCP/UDP 수신 확인
- `grep ':53'`는 `:5353`도 찾을 수 있으므로 **53과 5353을 혼동하지 말 것**

```bash
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --zone=public --add-service=dns
```

- 첫 줄: 실제 인터페이스에 적용된 firewalld Zone 확인(`ens160` → `public`이었음)
- 둘째 줄: `public` Zone에 DNS 서비스 런타임 허용
- **아직 영구 정책·재부팅 검증은 완료하지 않음**
- 실제 운영 환경은 허용 대상 네트워크/소스 주소까지 통제해야 함

## 5. 정방향/역방향 DNS 테스트

### `dig`

```bash
dig @127.0.0.1 web.lab.test A +short
dig @192.168.199.132 web.lab.test A +short
dig @192.168.199.132 -x 192.168.199.132 +short
```

- `@127.0.0.1`: 이 서버의 로컬 DNS에 직접 질의
- `@192.168.199.132`: 네트워크 주소에 바인딩된 DNS에 직접 질의
- `A`: 이름의 IPv4 주소 조회
- `-x`: 역방향(PTR) 질의 이름을 자동 생성
- `+short`: 출력 간결화

**실제 응답:** A → `192.168.199.132`, PTR → `web.lab.test.`

### Windows PowerShell

```powershell
Resolve-DnsName -Name web.lab.test -Type A -Server 192.168.199.132
nslookup 192.168.199.132 192.168.199.132
```

- `-Name`: 조회 이름
- `-Type A`: IPv4 주소 레코드
- `-Server`: 지정한 DNS 서버에 조회
- `nslookup <조회_IP> <서버_IP>`: 두 번째 주소가 **질의할 DNS 서버**, 첫 번째가 **역조회 대상**
- 결과가 성공했다고 Windows **기본 DNS 설정을 변경한 것은 아님**

## 6. 보안 진단·운영에서 혼동하면 안 되는 것

| 비교                            | 차이                                                                           |
| ------------------------------- | ------------------------------------------------------------------------------ |
| SSH 서버 상태 vs 포트           | `systemctl`로 프로세스 상태, `ss`로 실제 수신 소켓 확인                        |
| SSH 호스트 키 vs 사용자 인증 키 | 전자는 서버 검증, 후자는 사용자 인증                                           |
| passphrase vs 서버 비밀번호     | 전자는 로컬 개인키 보호, 후자는 서버 계정 비밀번호                             |
| 권한 DNS vs 재귀 DNS            | 권한 서버는 자신이 관리하는 Zone 데이터 응답, 재귀는 다른 DNS를 찾아 조회 수행 |
| A vs PTR                        | 이름→IP / IP→이름. 각각 별도 Zone·레코드                                       |
| named reload vs 재부팅          | 설정 재적용 / 운영체제 전체 재시작                                             |
| runtime vs permanent 방화벽     | 즉시 적용 / 재시작 이후도 유지하는 설정                                        |
| `named-checkzone` vs `dig`      | 구성 파일 검사 / 네트워크를 통한 실제 질의 결과 확인                           |

## 7. 빠른 연습용 명령 순서 (기존 환경에 손상 없음)

**Rocky:**

```bash
systemctl is-active sshd
sudo ss -tlnp | grep ':22'
sudo named-checkconf -z
sudo named-checkzone lab.test /var/named/lab.test.zone
sudo named-checkzone 199.168.192.in-addr.arpa /var/named/199.168.192.rev
dig @192.168.199.132 web.lab.test A +short
dig @192.168.199.132 -x 192.168.199.132 +short
```

**Windows:**

```powershell
Resolve-DnsName -Name web.lab.test -Type A -Server 192.168.199.132
nslookup 192.168.199.132 192.168.199.132
```

> 읽기 전용 확인 명령 위주다. 서비스·방화벽 보안 설정을 변경하는 명령은 준비 없이 실행하지 않는다.
