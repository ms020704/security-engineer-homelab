# Week 01 - Rocky Linux 서버 구축 및 기본 운영·보안

## 1. 목표

첫 주에는 Rocky Linux 서버를 직접 구축한 뒤, 단순 설치에서 끝내지 않고 아래 항목까지 실습하는 것을 목표로 했다.

- Rocky Linux 서버 구축
- 서버 기본 정보 및 네트워크 상태 확인
- Windows에서 SSH 원격 접속
- 사용자·그룹·파일 권한을 이용한 접근통제
- Apache HTTP Server 구축
- firewalld를 이용한 서비스 접근제어
- SSH/HTTP 서비스 장애 발생 및 복구
- 서비스 로그와 HTTP 접근 로그 확인

---

## 2. 실습 환경

| 항목 | 내용 |
|---|---|
| Host OS | Windows |
| Hypervisor | VMware Workstation 17 Player |
| Guest OS | Rocky Linux 10.2 |
| Hostname | `rocky-server01` |
| Network | VMware NAT |
| Server IP | `192.168.199.132/24` |
| Default Gateway | `192.168.199.2` |

> IP 주소는 VMware 내부 NAT 실습망의 사설 IP이다.

---

## 3. 전체 구성

```text
Windows Host
    |
    |  VMware NAT
    |
    +---- SSH / TCP 22 ----> Rocky Linux 10.2
    |
    +---- HTTP / TCP 80 ---> Apache httpd
```

---

## 4. 서버 기본 상태 확인

서버를 설치한 뒤 운영체제, 커널, 메모리, 디스크, 네트워크 정보를 확인했다.

```bash
whoami
hostnamectl
cat /etc/os-release
uname -r

free -h
df -h

ip addr
ip route
ping -c 4 8.8.8.8
ping -c 4 google.com
```

### 확인한 내용

- Rocky Linux 10.2가 정상 설치됨
- VMware NAT를 통해 외부 네트워크 통신 가능
- IP 통신뿐 아니라 DNS 이름 해석도 정상 동작
- 일반 사용자 `msk`가 `sudo`를 통해 관리자 권한 사용 가능

---

## 5. Hostname 설정

기본 `localhost` 대신 서버를 구분하기 쉽도록 hostname을 변경했다.

```bash
sudo hostnamectl set-hostname rocky-server01
hostnamectl
```

---

## 6. SSH 원격 관리

### SSH 서비스 확인

```bash
systemctl status sshd
sudo ss -tlnp
sudo firewall-cmd --list-all
```

확인 결과:

- `sshd.service` → `active (running)`
- TCP 22번 포트 → `LISTEN`
- firewalld → `ssh` 서비스 허용

### Windows에서 SSH 접속

```powershell
ssh msk@192.168.199.132
```

SSH 접속 후 다음 명령으로 실제 서버에 접속했는지 확인했다.

```bash
hostname
whoami
ip addr
```

---

## 7. SSH 장애 발생 및 복구

SSH 서비스를 의도적으로 중단했다.

```bash
sudo systemctl stop sshd
systemctl status sshd
```

이후 SSH 접속을 시도했을 때 다음 오류를 확인했다.

```text
Connection refused
```

서비스를 다시 시작해 복구했다.

```bash
sudo systemctl start sshd
```

### 배운 점

`Connection refused`는 서버까지는 도달했지만 해당 포트에서 연결을 받아줄 서비스가 없을 때 발생할 수 있다.

따라서 SSH 접속 장애가 발생하면 아래 순서로 확인할 수 있다.

```text
서버 통신 가능 여부
        ↓
sshd 서비스 상태
        ↓
TCP 22번 LISTEN 여부
        ↓
firewalld 정책
        ↓
인증 및 로그
```

---

## 8. Linux 기본 파일 관리

실습용 디렉터리를 만든 뒤 기본적인 파일 작업을 수행했다.

```bash
mkdir -p ~/security-lab/week01
cd ~/security-lab/week01

touch test.txt
echo "Security Engineer Lab - Week 01" > test.txt
cat test.txt

cp test.txt backup.txt
mv backup.txt backup-old.txt
ls -l
rm backup-old.txt
```

사용한 주요 명령:

- `mkdir` : 디렉터리 생성
- `cd` : 디렉터리 이동
- `pwd` : 현재 경로 확인
- `touch` : 빈 파일 생성
- `echo` : 문자열 출력 및 파일 작성
- `cat` : 파일 내용 확인
- `cp` : 파일 복사
- `mv` : 파일 이동/이름 변경
- `rm` : 파일 삭제
- `ls -l` : 파일 권한·소유자 등 상세 정보 확인

---

## 9. 사용자·그룹·파일 권한 접근통제

### 사용자와 그룹 생성

```bash
sudo useradd testuser
sudo passwd testuser

sudo groupadd security
sudo usermod -aG security testuser

id testuser
getent group security
```

### 중요 파일 생성 및 권한 설정

```bash
sudo mkdir /security
sudo touch /security/secret.txt

sudo chown root:security /security/secret.txt
sudo chmod 640 /security/secret.txt
```

확인:

```bash
ls -l /security
```

결과 형태:

```text
-rw-r-----. 1 root security ... secret.txt
```

### `640`의 의미

```text
Owner  : rw-  = 읽기 + 쓰기
Group  : r--  = 읽기
Others : ---  = 접근 불가
```

숫자 권한:

```text
r = 4
w = 2
x = 1
```

따라서:

```text
6 = 4 + 2 = rw-
4 = r--
0 = ---
```

### 권한 검증

`security` 그룹의 `testuser`:

```bash
su - testuser
cat /security/secret.txt
```

→ 읽기 성공

쓰기 시도:

```bash
echo "Hacked" >> /security/secret.txt
```

→ `Permission denied`

그룹에 속하지 않은 `outsider`:

```bash
sudo useradd outsider
sudo -u outsider cat /security/secret.txt
```

→ `Permission denied`

### 배운 점

Linux의 사용자·그룹·파일 권한을 이용해 다음과 같은 접근통제를 구현할 수 있었다.

```text
root           → 읽기 / 쓰기
security 그룹  → 읽기
그 외 사용자   → 접근 불가
```

---

## 10. Apache 웹 서버 구축

Apache HTTP Server를 설치했다.

```bash
sudo dnf install httpd -y
```

서비스를 즉시 시작하고 부팅 시 자동 시작하도록 설정했다.

```bash
sudo systemctl enable --now httpd
systemctl status httpd
```

TCP 80번 포트가 LISTEN 중인지 확인했다.

```bash
sudo ss -tlnp | grep ':80'
```

---

## 11. 웹 페이지 배포

간단한 테스트 페이지를 생성했다.

```bash
echo '<h1>Security Engineer HomeLab</h1>
<p>Week 01 - Rocky Linux Web Server</p>' \
| sudo tee /var/www/html/index.html
```

확인:

```bash
cat /var/www/html/index.html
```

Windows 브라우저에서 아래 주소로 접속했다.

```text
http://192.168.199.132
```

---

## 12. firewalld를 이용한 HTTP 접근제어

HTTP 서비스를 런타임 방화벽 정책에 추가했다.

```bash
sudo firewall-cmd --add-service=http
sudo firewall-cmd --list-services
```

### 방화벽 장애 실습

Apache를 실행한 상태에서 HTTP 허용 정책만 제거했다.

```bash
sudo firewall-cmd --remove-service=http
```

이 상태에서:

```bash
systemctl status httpd
sudo ss -tlnp | grep ':80'
```

결과는 다음과 같았다.

```text
httpd 서비스 : 정상
TCP 80       : LISTEN
외부 접속    : 실패
```

따라서 서비스 자체가 아니라 방화벽 정책 문제라고 판단할 수 있었다.

복구:

```bash
sudo firewall-cmd --add-service=http
```

> 참고: `--add-service=http`만 사용하면 runtime 설정이다. 영구 적용 시에는 `--permanent` 옵션과 `--reload`를 사용해야 한다.

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --reload
```

---

## 13. Apache 서비스 장애 실습

이번에는 방화벽 정책은 유지하고 Apache 자체를 중단했다.

```bash
sudo systemctl stop httpd
systemctl status httpd
sudo ss -tlnp | grep ':80'
```

결과:

```text
httpd 서비스 : inactive
TCP 80       : LISTEN 없음
외부 접속    : 실패
```

복구:

```bash
sudo systemctl start httpd
```

### 장애 비교

```text
[방화벽 문제]
httpd       : 실행 중
TCP 80      : LISTEN
외부 접속   : 실패

[서비스 문제]
httpd       : 중지
TCP 80      : LISTEN 없음
외부 접속   : 실패
```

같은 "웹 접속 실패"라도 서비스 상태와 포트 상태를 확인하면 원인을 구분할 수 있다는 것을 확인했다.

---

## 14. 로그 확인

### Apache 접근 로그

```bash
sudo tail -n 20 /var/log/httpd/access_log
```

Windows 브라우저에서 접속한 기록이 서버에 남는 것을 확인했다.

예:

```text
"GET / HTTP/1.1" 200
"GET /favicon.ico HTTP/1.1" 404
"GET / HTTP/1.1" 304
```

의미:

- `200` : 요청 정상 처리
- `404` : 요청한 파일 없음
- `304` : 브라우저 캐시 사용 가능

### Apache 서비스 로그

```bash
sudo journalctl -u httpd -n 30
```

Apache의 시작·중지·재시작 이력을 확인했다.

### SSH 로그

```bash
sudo journalctl -u sshd
```

SSH 로그인 성공과 서비스 중단/시작 기록을 확인했다.

---

## 15. 이번 주에 발생한 문제와 해결

### 문제 1. `dnf update` 권한 오류

일반 사용자 권한으로 실행:

```bash
dnf update
```

오류:

```text
This command has to be run with superuser privileges
```

해결:

```bash
sudo dnf update -y
```

---

### 문제 2. 일반 사용자의 `useradd` 실행 실패

```bash
useradd outsider
```

오류:

```text
Permission denied
cannot lock /etc/passwd
```

해결:

```bash
sudo useradd outsider
```

시스템 사용자 정보 변경에는 관리자 권한이 필요함을 확인했다.

---

### 문제 3. Windows PowerShell에서 Linux 명령어 실행

SSH 세션이 종료된 뒤 Windows PowerShell에서 `touch` 명령을 실행해 오류가 발생했다.

```text
PS C:\...>
```

와

```text
msk@rocky-server01:~$
```

프롬프트를 통해 현재 작업 환경이 Windows인지 Linux인지 구분해야 한다는 것을 배웠다.

---

## 16. 보안 관점에서 확인한 사항

열린 포트를 확인하면서 SSH와 HTTP 외에도 Cockpit(9090), CUPS(631) 관련 서비스가 존재하는 것을 확인했다.

```bash
sudo ss -tulpn
```

향후에는 사용하지 않는 서비스와 포트를 식별하고 제거하여 공격 표면을 줄이는 실습을 진행할 예정이다.

---

## 17. 이번 주에 배운 핵심

이번 실습을 통해 단순히 명령어를 암기하는 것보다 다음 흐름이 중요하다는 것을 배웠다.

```text
구축
 ↓
서비스 실행
 ↓
포트 확인
 ↓
방화벽 정책 확인
 ↓
접속 테스트
 ↓
장애 발생
 ↓
상태/포트/로그 분석
 ↓
복구
```

또한 Linux 서버 보안의 기본이 사용자·그룹·권한, 실행 중인 서비스, 열린 포트, 방화벽 정책, 로그 확인에서 시작된다는 것을 직접 확인했다.

---

## 18. 다음 학습 계획

다음 주에는 Week 01에서 만든 서버를 계속 사용하면서 다음 내용을 더 깊게 학습할 예정이다.

- 사용자 및 그룹 관리
- `sudo` 권한 관리
- 파일 및 디렉터리 권한
- 특수 권한
- 서비스 관리
- 불필요한 서비스 및 포트 점검
- 기본 서버 Hardening
