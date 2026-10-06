# Week 02 · Linux 시스템 관리 기본기

> **교재 기반 학습:** 『이것이 리눅스다 with Rocky Linux 10』 Chapter 4  
> **학습 방식:** 교재로 개념 이해 → Rocky Linux에서 명령 실행 → 결과 해석 → 내 말로 설명

리눅스 서버 운영에 필요한 기본 명령부터 사용자 권한, 네트워크, 서비스, 부팅 구조까지 연결해 정리한다. 보안 엔지니어의 기초 역량인 **권한 확인, 서비스 상태 점검, 로그 기반 문제 분석**에 초점을 둔다.

이 문서는 4장 학습 정리와 재현 가능한 실습 가이드다.

## 1. 학습 목표

- 기본 명령과 도움말을 이용해 파일·디렉터리를 관리한다.
- 사용자·그룹·권한을 설명하고 최소한의 접근 권한을 설정한다.
- 패키지, 예약 작업, 프로세스와 서비스를 점검한다.
- 접속 문제를 IP → 경로 → 서비스 → 포트 → 방화벽 → 로그 순서로 분석한다.
- GRUB, 커널, systemd의 역할과 복구 모드의 차이를 설명한다.

## 2. 실습 환경

| 항목 | 기준 / 확인 방법 |
| --- | --- |
| 대상 OS | Rocky Linux 10 — 실제 설치 버전은 `cat /etc/os-release`로 확인 |
| 셸 | Bash 기준 — `echo "$SHELL"`로 로그인 셸 확인 |
| 권한 | 일반 사용자로 진행하고 관리 작업에만 `sudo` 사용 |
| 작업 위치 | `~/ch4-lab` |
| 환경 기록 | `uname -r`, `systemctl get-default`, `ip -br addr` 출력 기록 |

```bash
mkdir -p ~/ch4-lab
cd ~/ch4-lab
cat /etc/os-release
uname -r
systemctl get-default
```

종료·재부팅·복구 모드 전환은 세션을 중단하므로 별도 실습 시 수행한다. 장치명과 네트워크 연결 이름은 실제 환경에서 확인해 사용한다.

## 3. 범위별 실습 명령어

아래 표는 **실행·설명할 명령과 확인 기준**이다. 설치되지 않은 도구는 패키지 제공 여부를 먼저 확인한다.

| 학습 범위 | 대표 명령 / 조작 | 확인할 내용 |
| --- | --- | --- |
| 시작·종료·재부팅·로그아웃 | `journalctl -b`, `sudo shutdown now`, `sudo reboot`, `logout` | 부팅 로그, 시스템 종료와 사용자 세션 종료의 차이. `logout`은 로그인 셸에서 사용하며 일반 셸은 `exit` |
| 가상 콘솔 | `Ctrl+Alt+F3`, `tty` | 별도 TTY 로그인과 현재 터미널. 키 전달·GUI 복귀 키는 환경에 따라 다름 |
| 기본 명령 | `pwd`, `ls -al`, `cd`, `mkdir`, `touch`, `cp`, `mv`, `cat`, `rm` | 경로, 숨김 파일, 복사·이동·삭제 결과 |
| 런레벨·target | `systemctl get-default`, `systemctl list-units --type=target` | 기본 부팅 목표와 현재 활성 target |
| 히스토리·자동완성 | `history`, `!번호`, `Tab` | 명령 재사용. `!번호`는 해당 명령을 확인한 뒤 실행 |
| 편집기·도움말 | `vi notes.txt`, `man chmod`, `ls --help` | vi의 `i` 입력, `Esc` 명령 모드, `:wq` 저장·종료, `:q!` 변경 폐기; man 종료는 `q` |
| 마운트 | `lsblk -f`, `findmnt`, `df -h`, `mount`, `umount` | 장치·파일시스템·마운트 지점의 관계, `/etc/fstab`의 영구 설정 역할 |
| 사용자·그룹 | `id`, `getent passwd`, `getent group`, `useradd`, `passwd`, `groupadd`, `usermod -aG` | UID/GID, 기본·보조 그룹. 보조 그룹 변경은 새 로그인에서 확인 |
| 소유권·권한 | `ls -l`, `chmod 640 파일`, `chown 사용자:그룹 파일`, `umask` | owner/group/others, 접근 허용 범위, 생성 시 권한 제한 |
| RPM·DNF | `rpm -q bash`, `rpm -ql bash`, `dnf info tree`, `sudo dnf install tree`, `sudo dnf remove tree` | 설치 여부·패키지 파일 목록·의존성 처리. 변경 전 트랜잭션 확인 |
| 압축·검색 | `tar -czf`, `tar -tzf`, `tar -xzf`, `find`, `which`, `whereis` | 아카이브 생성·목록·해제, 조건 검색, 실행 파일·매뉴얼 위치 |
| 시스템 설정 | `hostnamectl`, `timedatectl`, `localectl`, `hostnamectl --help` | 호스트명·시간대·로케일의 현재 값과 변경 방법 |
| 예약 작업 | `crontab -e`, `crontab -l`, `at`, `atq`, `atrm` | 반복 작업과 일회성 작업, `crond`·`atd` 상태 |
| 네트워크 | `ip -br addr`, `ip route`, `nmcli device status`, `nmcli connection show`, `getent hosts example.com` | 주소·게이트웨이·연결 프로필·이름 해석 |
| 파이프·필터·리디렉션 | `grep`, `sort`, `uniq`, `wc`, `head`, `tail`, `\|`, `>`, `>>`, `2>` | 데이터 선별과 표준 입력·출력·오류의 흐름 |
| 프로세스 | `ps aux`, `top`, `pgrep`, `jobs`, `fg`, `bg`, `kill` | PID, 자원 사용, 작업 제어와 종료 신호 |
| 서비스·로그 | `systemctl status sshd`, `systemctl is-enabled sshd`, `journalctl -u sshd -b` | 현재 실행 상태, 부팅 시 활성화 설정, 해당 부팅의 서비스 로그 |
| systemd 서비스·소켓 | `systemctl cat sshd.service`, `systemctl list-sockets --all` | 유닛 설정과 소켓 활성화 구조. 서비스마다 소켓 유닛이 있는 것은 아님 |
| 복구·긴급 모드 | `systemctl cat rescue.target emergency.target`, `systemctl --failed` | 복구 환경의 차이와 실패 유닛 확인 |
| GRUB·부팅 | `cat /proc/cmdline`, `sudo grubby --info=ALL` | 현재 커널 인자와 등록된 부팅 항목. GRUB 설정 변경은 별도 실습 |
| 커널·모듈 | `uname -r`, `lsmod`, `modinfo 모듈명`, `modprobe --help` | 실행 중인 커널, 로드된 모듈, 모듈 정보와 관리 방법 |

## 4. 핵심 실습과 확인 기준

### A. 파일·권한·압축·검색

```bash
cd ~/ch4-lab
mkdir -p files restored
printf 'linux\nsecurity\nlinux\n' > files/notes.txt
cp files/notes.txt files/copy.txt
mv files/copy.txt files/renamed.txt
chmod 640 files/notes.txt
ls -l files/notes.txt

tar -czf files.tar.gz files
tar -tzf files.tar.gz
tar -xzf files.tar.gz -C restored
diff files/notes.txt restored/files/notes.txt
find ./files -type f -name '*.txt'
which bash
whereis bash
```

**확인:** `640`은 소유자 읽기·쓰기, 그룹 읽기, 기타 사용자 접근 불가다. `diff`가 출력 없이 종료 코드 0을 반환하면 두 파일의 내용이 같다. 상위 디렉터리 접근 권한도 파일 접근에 영향을 준다.

### B. 사용자·그룹과 소유권

별도 실습 계정 이름이 기존에 없는지 `getent passwd ch4user`, `getent group ch4team`으로 먼저 확인한다.

```bash
sudo groupadd ch4team
sudo useradd -m -G ch4team ch4user
sudo passwd ch4user
id ch4user
sudo touch /home/ch4user/shared.txt
sudo chown ch4user:ch4team /home/ch4user/shared.txt
sudo chmod 640 /home/ch4user/shared.txt
sudo -u ch4user test -w /home/ch4user/shared.txt && echo 'owner: writable'
sudo ls -l /home/ch4user/shared.txt
```

**확인:** 계정의 기본 그룹과 보조 그룹을 구별하고, 파일 소유자·그룹·권한을 함께 읽는다. `usermod -aG`의 `-a`는 기존 보조 그룹을 유지하며 추가한다.

### C. 파이프·리디렉션·프로세스

```bash
cd ~/ch4-lab
sort files/notes.txt | uniq -c
grep 'linux' files/notes.txt | wc -l
printf 'review\n' >> files/notes.txt
ls ./no-such-ch4-file 2> error.log
cat error.log

sleep 300 &
lab_pid=$!
ps -p "$lab_pid" -o pid,comm,stat
kill "$lab_pid"
```

**확인:** `grep` 결과는 2줄이다. `>`는 덮어쓰기, `>>`는 추가, `2>`는 오류 출력 저장이다. `|`는 앞 명령의 표준 출력을 다음 명령의 표준 입력으로 전달한다. 기본 `kill`은 SIGTERM으로 종료를 요청한다.

### D. 마운트·예약 작업

- **마운트:** `lsblk -f`로 실습 장치를 확인한 뒤 `sudo mount /dev/실습장치 /mnt/ch4` 형태로 연결하고 `findmnt /mnt/ch4`로 확인한다. 마운트 지점은 먼저 생성하며, 작업 후 다른 경로로 이동해 `sudo umount /mnt/ch4`로 해제한다. 예시 장치명은 실제 값으로 교체한다.
- **cron:** `crontab -e`에 `0 18 * * * /usr/bin/date >> "$HOME/ch4-lab/cron.log"`를 등록하면 매일 시스템 시간 기준 18시에 실행된다. 필드는 **분·시·일·월·요일** 순서이며 `crontab -l`로 확인한다.
- **at:** `echo 'date >> "$HOME/ch4-lab/at.log"' | at now + 2 minutes`로 일회성 작업을 예약하고 `atq`로 확인한다. 취소는 `atrm 작업번호`를 사용한다.

예약 작업은 관련 패키지와 `crond`·`atd` 서비스가 필요하다. 로그 생성까지 확인한 뒤 테스트용 cron 항목만 편집해 제거한다. 예약 작업의 서비스·설정 설명은 [Rocky Linux 공식 문서](https://docs.rockylinux.org/es/books/admin_guide/11-tasks/)를 참고한다.

## 5. 반드시 설명할 핵심 개념

| 비교 | 핵심 설명 |
| --- | --- |
| `sudo` / root | `sudo`는 정책이 허용하는 명령을 다른 사용자 권한으로 실행한다. 기본 대상은 root이며 모든 사용자에게 허용되는 것은 아니다. |
| 파일 / 디렉터리 권한 | 파일의 `r/w/x`는 읽기·쓰기·실행, 디렉터리의 `r/w/x`는 목록 조회·항목 생성/삭제·탐색이다. 삭제에는 부모 디렉터리 권한 등이 영향을 준다. |
| RPM / DNF | RPM은 패키지 설치·조회 등 저수준 관리 도구다. DNF는 저장소를 이용하고 필요한 의존 패키지를 함께 해결한다. |
| 프로세스 / 서비스 | 프로세스는 실행 중인 프로그램 인스턴스다. 서비스는 systemd 등이 관리하는 기능 단위이며 하나 이상의 프로세스를 포함할 수 있다. |
| `start` / `enable` | `start`는 지금 시작, `enable`은 부팅 등에서 시작되도록 연결 설정이다. 지원하는 유닛에서 `enable --now`는 둘을 함께 수행한다. |
| service / socket | service 유닛은 실행 동작을 정의한다. socket 유닛은 소켓을 준비하고 요청에 따라 연결된 서비스를 활성화할 수 있다. |
| 런레벨 / target | 기존 런레벨 3·5는 대체로 `multi-user.target`·`graphical.target`에 대응한다. target은 여러 유닛을 묶는 목표이며 단순한 숫자 단계가 아니다. |
| rescue / emergency | rescue는 복구에 필요한 기본 서비스와 로컬 파일시스템을 준비한다. emergency는 더 최소한의 복구 환경이다. |
| 커널 / 모듈 | 커널은 CPU·메모리·장치 등 자원을 관리한다. 모듈은 필요에 따라 로드·해제할 수 있는 커널 기능이다. |

부팅의 큰 흐름은 **펌웨어 → GRUB → 커널·initramfs → 실제 루트 파일시스템 → systemd → 서비스·로그인 환경**이다. systemd와 복구 모드의 세부 동작은 [Rocky Linux 시스템 시작 문서](https://docs.rockylinux.org/ko/books/admin_guide/10-boot/)를 참고한다.

## 6. 트러블슈팅: SSH 접속이 안 될 때

**IP 확인 → 라우팅 확인 → 서비스 확인 → LISTEN 확인 → 방화벽 확인 → 로그 분석 → 재시험**

| 순서 | 확인 명령 | 판단 기준 |
| --- | --- | --- |
| 1. 주소·이름 해석 | `ip -br addr`, `getent hosts 서버이름` | 대상 IP와 인터페이스 상태, 이름이 올바른 주소로 해석되는지 |
| 2. 경로 | `ip route`, `ping -c 3 대상IP` | 경로와 통신 가능성. ping 실패만으로 서버 장애를 확정하지 않음 |
| 3. 서비스 | `systemctl status sshd` | `active`인지, 시작 실패 메시지가 있는지 |
| 4. 실제 포트 | `sudo ss -lntp` | 설정된 SSH 포트와 바인딩 주소에서 LISTEN 중인지 |
| 5. 접근 통제 | `sudo firewall-cmd --get-active-zones`, `sudo firewall-cmd --list-all --zone=존이름` | 인터페이스에 적용된 존, 서비스·포트 허용 여부. 존 이름은 실제 값 사용 |
| 6. 로그 | `sudo journalctl -u sshd -b --no-pager -n 50` | 설정·인증·권한 오류 원인 확인 |

방화벽 단계는 firewalld를 사용하는 환경 기준이다. 필요하면 VM 네트워크 모드와 외부 방화벽도 확인한다. SELinux 거부가 의심되면 `getenforce`, `sudo ausearch -m AVC -ts recent`로 근거를 확인한다. 서비스가 실행 중이어도 포트·경로·인증 문제로 접속은 실패할 수 있다.

실제 장애 기록은 **증상 → 관찰한 출력 → 원인 → 수정 → 재시험 결과** 순서로 남긴다.

## 7. 배운 점과 검증

이번 범위의 핵심은 명령어를 개별적으로 외우는 데서 나아가 **권한·프로세스·서비스·네트워크·로그를 연결해 판단하는 것**이다. 정리한 주요 학습 포인트는 다음과 같다.

- 파일 접근 문제는 권한 숫자뿐 아니라 소유자·그룹과 상위 경로까지 함께 봐야 한다.
- 서비스의 현재 상태와 자동 시작 설정은 별도로 확인해야 한다.
- 명령 실행 뒤에는 출력·종료 상태·로그를 확인해야 작업 성공을 판단할 수 있다.
- 부팅과 복구 구조를 이해하면 문제가 생긴 단계를 좁힐 수 있다.

### 직접 설명하고 재현할 수 있는지 확인

- [ ] `chmod 640`을 설명하고 설정 결과를 보여준다.
- [ ] 사용자 기본 그룹과 보조 그룹을 구분한다.
- [ ] RPM과 DNF의 역할을 비교한다.
- [ ] tar로 압축·해제하고 원본과 내용을 비교한다.
- [ ] 파이프와 리디렉션의 차이를 예시로 설명한다.
- [ ] `start`와 `enable`, 서비스와 소켓의 차이를 설명한다.
- [ ] 접속 문제를 명령 출력과 로그로 단계적으로 확인한다.
- [ ] GRUB·커널·systemd와 rescue·emergency의 역할을 설명한다.


**학습 자료:** 『이것이 리눅스다 with Rocky Linux 10』 Chapter 4를 범위 기준으로 삼고, 명령의 세부 옵션은 실습 환경의 `man`·`--help`와 Rocky Linux 공식 문서로 확인한다. AI는 개념 정리와 복습 보조에 활용하며, 실습 완료 여부는 직접 확인한 결과로 기록한다.
