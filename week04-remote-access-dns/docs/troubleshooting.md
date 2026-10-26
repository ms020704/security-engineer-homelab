# Week04 트러블슈팅 — 실제 관찰과 아직 겪지 않은 상황 구분

[메인 README](../README.md) · [명령어 사전](command-reference.md)

## 1. 실제로 확인한 실습 결과 및 주의할 설정

| 항목 | 실제 관찰 | 결론 및 유의점 |
|---|---|---|
| `systemctl is-enable sshd` | 알 수 없는 동사라는 오류 | 올바른 명령은 `systemctl is-enabled sshd`. **오타 문제**, sshd 장애 아님 |
| `sshd` 상태 | `active`, `enabled`, TCP 22 LISTEN | 접속 준비 완료. 실행·부팅 활성화·포트는 각각 따로 확인 |
| SSH 현재 정책 | `permitrootlogin yes`, `passwordauthentication yes`, `pubkeyauthentication yes` | 공개키 인증은 성공했지만 보안 강화는 **아직 적용하지 않음** |
| 공개키 인증 | Windows에서 개인키 passphrase 입력 후 `msk`, `rocky-server01` 출력 | `PasswordAuthentication=no` 옵션으로 공개키 인증 사용 확인 |
| SCP | `week04.txt` 100% 전송 후 Rocky에서 내용 확인 | 단순 전송 표시뿐 아니라 서버 실제 파일 읽기 검증 |
| `named-checkconf` | 검사 후 오류 없이 완료 | 구성 파일 문법 점검 통과 |
| `named-checkzone` | 정방향·역방향 Zone `loaded serial ...` 및 `OK` | RR 파일 검증 성공 |
| BIND 수신 | `127.0.0.1:53`, `192.168.199.132:53` UDP/TCP | 네트워크 DNS 질의 받을 준비 됨 |
| 방화벽 | `public`, `ens160`, `--add-service=dns` 성공 | 런타임 허용, 영구 정책과 재부팅 검증 별개 |
| Linux DNS 조회 | A·PTR 올바른 응답 | named와 Zone 질의 성공 |
| Windows DNS 조회 | A·PTR 올바른 응답 | 네트워크를 넘는 **클라이언트 검증** 성공 |

## 2. 장애 진단은 계층별로

```text
[Windows에서 SSH/DNS 안 됨]
    ↓ 1) 대상 서버 IP·VMware NAT·인터페이스 UP?
    ↓ 2) service 상태 active? (`systemctl`)
    ↓ 3) 포트 LISTEN? (`ss`)
    ↓ 4) 방화벽·네트워크 경로 정상? (`firewall-cmd`, 라우팅)
    ↓ 5) 애플리케이션 ACL/인증 설정? (`sshd -T`, named.conf)
    ↓ 6) 데이터·계정·Zone 내용 정상? (authorized_keys, named-checkzone)
    ↓ 7) journal 확인 후 다시 접속 테스트
```

**원칙:** 접속이 안 된다는 증상 하나로 원인을 단정하지 말고 각 계층을 점검한다.

## 3. SSH 장애별 확인 방법 — 추가 학습용

### 연결 거부(`Connection refused`)

- 먼저 대상 IP와 포트가 맞는지 확인
- `systemctl status sshd`, `sudo ss -tlnp | grep ':22'`
- 서비스가 중단되어 있거나 다른 주소/포트에 바인딩되었을 수 있음

### 접속 시간 초과(`Connection timed out`)

- 방화벽, VM 네트워크, 라우팅, IP, 중간 장비 등을 확인
- 시간 초과만으로 서비스 프로세스의 정상/비정상을 단정하지 않음

### 공개키 인증 실패(`Permission denied (publickey)`)

- 연결하려는 사용자명과 홈 디렉터리가 맞는지
- 서버 계정의 `~/.ssh/authorized_keys`에 올바른 공개키가 등록되었는지
- `~/.ssh`와 `authorized_keys` 권한이 안전한지
- 클라이언트가 `-i`로 올바른 개인키를 사용하는지
- `journalctl -u sshd`에서 실제 실패 이유 확인

### 서버 호스트 키 변경 경고

- 서버 재설치 등 정상 사유인지 별도 채널로 지문 확인
- 정체가 확인되기 전에는 `known_hosts`를 무작정 삭제해 경고를 우회하지 않음

## 4. DNS 장애별 확인 방법 — 추가 학습용

| 현상 | 먼저 확인할 것 | 대표 명령 |
|---|---|---|
| localhost 질의 실패 | named 상태, Zone 문법, 로그 | `systemctl status named`; `named-checkconf -z` |
| localhost OK / Windows 실패 | 바인딩 주소, UDP/TCP 53, firewalld, `allow-query`, NAT | `ss -lntup`; `firewall-cmd --get-active-zones` |
| 정방향 OK / 역방향 실패 | reverse Zone·PTR 레코드·Zone 선언 | `named-checkzone 199.168.192.in-addr.arpa ...` |
| 기존 값만 조회됨 | TTL 캐시, Serial, named reload, 조회 대상 서버 | `dig @192.168.199.132 ...` |
| `REFUSED` | DNS ACL·allow-query, 재귀 정책 | `/etc/named.conf` 점검 |
| `NXDOMAIN` | 해당 이름 레코드 부재, Zone 이름 오타 | `dig` 전체 결과 확인 |
| `SERVFAIL` | 서버 처리 오류·위임·DNSSEC·설정 등 | `journalctl -u named` |

**중요 구별:** `ss -lntup | grep ':53'`에 나타날 수 있는 **mDNS UDP 5353(avahi-daemon)**는 DNS 53(named)과 다른 서비스다.

## 5. 자주 하는 오해

1. **"A 레코드가 됐으니 DNS 구축이 완벽히 끝났다"** → 권한 Zone의 해당 조회만 성공한 것. 접근 정책·로그·재부팅은 별도 검증.
2. **"Windows에서 조회됐으니 Windows 기본 DNS가 바뀌었다"** → `-Server` 옵션은 특정 DNS 서버를 직접 지정할 뿐임.
3. **"PTR로 서버 신원이 증명된다"** → PTR은 DNS 이름 정보로 인증 수단이 아님.
4. **"SSH 키 passphrase는 서버 비밀번호다"** → 개인키 잠금 해제 문구와 Linux 계정 암호는 다름.
5. **"`permitrootlogin yes`여도 공개키 인증이 되니 안전하다"** → 어떤 사용자와 인증이 허용되는지 별도 보안 검토 필요.
6. **"DNS는 UDP만 쓴다"** → TCP 53도 필요.
7. **"`firewall-cmd --add-service=dns`는 재부팅 뒤에도 영구 적용된다"** → 런타임 설정이며 영구 정책과 다름.

## 6. 아직 검증하지 않은 작업

- [ ] sshd root 원격 로그인 제한 및 비밀번호 인증 비활성화 이후 재접속 시험
- [ ] DNS 실효 ACL·재귀 거부 상태 검증
- [ ] DNS 포트 방화벽을 필요한 사설 네트워크로 한정한 영구 규칙
- [ ] DNS 정책 재부팅 유지 확인
- [ ] 그래픽 원격 접속 XRDP/GNOME RDP 실제 테스트
- [ ] 일부러 Zone RR을 잘못 적은 후 `named-checkzone` 진단 실습(별도 안전 실습 환경에서)
