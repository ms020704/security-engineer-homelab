# Chapter 8 — 원격지 시스템 관리: Telnet·OpenSSH·원격 GUI

> **목표:** 원격 관리에서 무엇을 신뢰해야 하고, 어떤 인증·암호화 과정이 일어나는지 설명한다. 실제 실습은 SSH/공개키 인증/SCP까지 완료했다. Telnet 서버 구축과 그래픽 원격 접속은 완료하지 않았다.

[메인 README로 돌아가기](../README.md)

## 1. 원격지 서버 관리가 필요한 이유

실제 서버는 데이터센터, 기업 서버실, 원격 클라우드에 있기 때문에 서버 앞에 직접 앉지 않고 네트워크로 상태를 확인하고 설정을 변경하는 일이 많다. 원격 관리에는 **접속 기능**뿐 아니라 **누구인지 인증**, **전송 중 보호**, **허용 범위 통제**, **기록 및 사고 추적**이 중요하다.

| 항목   | 의미                          | 진단 질문                         |
| ------ | ----------------------------- | --------------------------------- |
| 가용성 | 관리용 서비스가 동작하는가?   | sshd가 active인가?                |
| 연결성 | 네트워크와 포트가 연결되는가? | 서버 IP와 TCP 22 LISTEN 상태는?   |
| 인증   | 올바른 사용자만 접속하는가?   | 공개키 인증에 성공했나?           |
| 권한   | 접속 후 허용된 작업만 하는가? | sudo 권한은 필요한 만큼만 주었나? |
| 추적성 | 누가 언제 무엇을 했나?        | `journalctl -u sshd`, 로그인 로그 |

## 2. Telnet vs SSH

| 비교      | Telnet                                 | SSH                             |
| --------- | -------------------------------------- | ------------------------------- |
| 대표 포트 | TCP 23                                 | TCP 22                          |
| 전송 보호 | 기본적으로 암호화 없음                 | 암호화된 채널 제공              |
| 주요 용도 | 레거시 원격 터미널, 제한된 진단        | 서버 원격 셸, 파일 전송, 터널링 |
| 실무 권장 | 신뢰할 수 없는 망에서 로그인 사용 금지 | 원격 관리 표준적 선택           |

- **Telnet은 모든 네트워크 용도로 금지해야 하는 하나의 악성 프로그램이라는 뜻이 아니다.** 문제는 원격 로그인에서 민감한 내용을 보호하지 못한다는 데 있다.
- SSH는 암호화뿐 아니라 **접속한 서버가 올바른 서버인지**와 **접속자가 권한 있는 사용자인지** 확인할 수 있다.
- 이번에는 **Telnet 서버를 실제로 실행하지 않고** 보안 차이를 개념으로 비교했다.

## 3. SSH 연결의 주요 과정

```text
Windows SSH 클라이언트
  ↓ TCP 22 연결
Rocky sshd 서버
  ↓ 프로토콜/암호 알고리즘 협상 + 키 교환
암호화된 세션 수립
  ↓ 서버 호스트 키(서버 신원) 검증
서버 신원 확인
  ↓ 사용자 인증(비밀번호 또는 공개키 등)
로그인 허용 → 암호화된 셸 / 파일 전송
```

### (1) 서버 호스트 키 vs 사용자 인증 키

| 구분                      | 서버 호스트 키                                     | 사용자 SSH 키                                        |
| ------------------------- | -------------------------------------------------- | ---------------------------------------------------- |
| 누구의 신원을 확인?       | 서버                                               | 접속하려는 사용자                                    |
| 개인키 보관 위치          | SSH **서버**                                       | 사용자의 **클라이언트**(Windows)                     |
| 클라이언트/서버 등록 정보 | 클라이언트의 `known_hosts`가 서버 호스트 키를 기억 | 서버 계정의 `authorized_keys`가 사용자 공개키를 허용 |
| 핵심 질문                 | '이 서버가 내가 접속하려던 서버인가?'              | '이 접속자가 허용된 사용자인가?'                     |

**중요:** '공개키로 데이터를 암호화하고 개인키로 푼다'라는 단순 표현만으로 SSH 인증 전부를 설명하지 않는다. SSH 공개키 사용자 인증은 **클라이언트가 개인키를 사용해 서명하고 서버가 등록된 공개키로 검증**하는 방식이다. 세션 통신 암호화 키는 별도의 키 교환을 통해 형성된다.

### (2) passphrase vs Linux 로그인 비밀번호

- **키 passphrase**: Windows의 개인키 파일을 보호하는 암호문구. 화면에 `Enter passphrase for key`라고 표시된다.
- **Linux 계정 비밀번호**: Rocky 서버의 사용자 계정 인증에 사용되는 비밀번호. 공개키 인증만 시도하면 사용하지 않을 수 있다.
- 서버가 개인키 내용을 받아서 보관하는 구조는 아니다. 서버에는 **공개키**가 등록된다.

### (3) 왜 첫 접속 때 fingerprint를 확인하는가?

서버 호스트 키가 이전에 알려져 있지 않으면 사용자가 서버의 키 지문을 확인하라는 경고를 볼 수 있다. 신뢰할 수 있는 경로로 지문을 검증하지 않고 무조건 `yes`를 누르면 서버 사칭 공격을 탐지하지 못할 수 있다.

## 4. 서버에서 확인했던 명령어

```bash
hostname -I
systemctl is-active sshd
systemctl is-enabled sshd
sudo ss -tlnp | grep ':22'
sudo sshd -T | grep -E '^(port|pubkeyauthentication|passwordauthentication|permitrootlogin) '
```

| 명령어            | '무엇을 보는가'               | '왜 필요한가'                               |
| ----------------- | ----------------------------- | ------------------------------------------- |
| `hostname -I`     | 호스트의 IP 목록              | 정확한 접속 주소 확인                       |
| `is-active sshd`  | SSH 서비스 **현재** 실행 상태 | 동작하지 않으면 원격 접속 불가              |
| `is-enabled sshd` | 부팅 시 서비스 시작 설정      | 현재 실행과 부팅 연동은 별개                |
| `ss -tlnp`        | TCP LISTEN 및 프로세스        | 실제 22번 포트에서 수신하는지 확인          |
| `sshd -T`         | 데몬 적용 설정 출력           | 단순히 파일에 적힌 설정이 아닌 실효 값 점검 |

실제 출력에서 `active`, `enabled`, `0.0.0.0:22`, `[::]:22`와 다음을 확인했다.

```text
port 22
permitrootlogin yes
pubkeyauthentication yes
passwordauthentication yes
```

**보안 미완료:** `permitrootlogin yes`와 `passwordauthentication yes`를 발견했을 뿐, 이를 안전한 값으로 변경한 후 검증한 기록은 없다. 또한 Match 블록이 있는 구성에서는 `sshd -T -C ...` 등 접속 문맥에 따라 효과적 설정이 달라질 수 있다.

## 5. ED25519 공개키 인증 실습 기록

### Windows PowerShell (실제 사용한 명령의 재현 예시)

```powershell
# 기존 파일이 존재하는지 먼저 확인해야 함
Test-Path "$env:USERPROFILE\.ssh\id_ed25519_rocky_lab"
ssh-keygen -t ed25519 -f "$env:USERPROFILE\.ssh\id_ed25519_rocky_lab" -C "rocky-lab"
$ip = "192.168.199.132"
$server = "msk@$ip"

Get-Content "$env:USERPROFILE\.ssh\id_ed25519_rocky_lab.pub" |
  ssh $server "umask 077; mkdir -p ~/.ssh; cat >> ~/.ssh/authorized_keys"

ssh -i "$env:USERPROFILE\.ssh\id_ed25519_rocky_lab" `
  -o PreferredAuthentications=publickey `
  -o PasswordAuthentication=no $server "whoami; hostname"
```

실제 결과: `msk`, `rocky-server01` 출력, 공개키 인증 테스트 성공.

### `authorized_keys`가 하는 일

- 해당 Linux 사용자 계정의 **허용된 공개키** 목록이다.
- `~/.ssh` 디렉터리와 `authorized_keys`는 다른 사용자에게 쓰기 가능하게 열어놓지 않아야 한다.
- 이번 등록 과정에서는 `umask 077`을 사용했다. 권한은 다음 명령으로 점검할 수 있다.

```bash
ls -ld ~/.ssh
ls -l ~/.ssh/authorized_keys
```

_위 두 확인 명령은 권장 복습 명령이며 실제 실행 화면은 별도로 수집하지 않았다._

## 6. SCP 파일 전송 실습

```powershell
Set-Content -Path "$env:TEMP\week04.txt" -Value "SSH file transfer test"
scp -i "$env:USERPROFILE\.ssh\id_ed25519_rocky_lab" `
  "$env:TEMP\week04.txt" "${server}:/home/msk/week04.txt"
```

Rocky Linux에서 다음을 실행해 파일 내용까지 확인했다.

```bash
cat ~/week04.txt
# SSH file transfer test
```

### SCP vs SFTP

- `scp`는 파일 복사를 위한 명령줄 클라이언트다. 현대 OpenSSH의 `scp`는 기본적으로 SFTP 프로토콜을 이용할 수 있다(구현과 옵션에 따라 다름).
- `sftp`는 파일 나열, 전송, 이동 등 파일 관리에 쓰이는 별도의 클라이언트다.
- 두 경우 모두 SSH 기반 연결과 인증을 활용할 수 있다. **이번에 직접 검증한 것은 `scp`**이다.

## 7. 원격 접속 보안 개선 방향 (아직 적용되지 않은 것)

1. 개인키에 passphrase 설정, OS 계정에 맞는 보관 권한, 키 분실·유출 대응.
2. 가능하면 일반 계정으로 SSH 로그인하고 필요한 명령만 `sudo`로 관리.
3. 별도 SSH 연결을 유지한 채 `sshd -t`로 구문 확인한 뒤 설정 변경.
4. 키 인증으로 다른 세션에서 접속 성공 여부를 확인한 후 비밀번호 인증 비활성화 여부 검토.
5. root의 원격 로그인 정책은 업무·구성에 맞게 제한.
6. 네트워크 방화벽에서 관리 클라이언트 IP만 허용하고 외부 포트 노출 최소화.
7. `journalctl -u sshd`와 `last` 등을 통해 인증·접속 기록 점검.

⚠️ SSH를 원격으로 사용 중일 때 설정을 일괄 변경하면 관리 접속 자체를 잃을 수 있다. VMware 콘솔 복구 경로를 확보하고 변경하자.

## 8. Chapter 8.3 — XRDP와 GNOME RDP

- 책에서 설명하는 **XRDP**는 Linux에서 RDP 서버 기능을 제공하는 별도 구현이다.
- Rocky Linux 10에서는 **GNOME Remote Desktop**을 통해 RDP 원격 공유/로그인을 구성할 수 있다. XRDP와 동일한 패키지는 아니다.
- 실제 Windows 원격 데스크톱 연결(`mstsc`)을 성공시킨 화면은 없으므로 **이번 장 전체 실습 완료로 표기하지 않는다**.
- 원격 데스크톱을 테스트할 때는 VMware 사설망으로만 제한하고 필요한 포트만 확인하여 허용한다.

## 9. 간단 요약

> Chapter 8에서는 기존 SSH 서버의 상태·포트·설정을 확인한 뒤 Windows에서 ED25519 공개키 인증을 구성했다. SSH는 서버 호스트 키를 이용해 서버를 검증하고, 사용자 인증에서는 등록한 공개키와 클라이언트 개인키를 이용한다. 비밀번호 인증을 허용하지 않는 옵션으로 키 인증을 검증했고 SCP를 이용한 파일 전송도 성공했다. 텔넷은 암호화가 제공되지 않아 SSH와 보안상 차이가 있고, 원격 GUI 접속은 아직 별도로 검증하지 않았다.

## 10. 책 덮고 복습

- 왜 SSH에서 **호스트 키**와 **사용자 키**가 서로 다른가?
- 공개키를 `authorized_keys`에 등록할 때 개인키를 서버로 복사하면 왜 안 되는가?
- 키 passphrase를 입력했는데도 공개키 인증이라고 말할 수 있는 이유는?
- `sshd -T`는 무엇을 보여주고 `ss`와 어떻게 다른가?
- `permitrootlogin yes`가 의미하는 위험은 무엇인가?
- 원격으로 SSH 보안 설정을 바꿀 때 접속이 끊기는 문제를 어떻게 예방할까?

[복습 문제의 해설 보기](review-questions.md)
