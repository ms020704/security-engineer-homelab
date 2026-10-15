# Week 03 — Rocky Linux 디스크 관리·Quota·RAID·Bash 자동화

> **기준 교재:** 『이것이 리눅스다 with Rocky Linux 10』 Chapter 6(디스크 관리·사용자별 공간 할당), Chapter 7(셸 스크립트 프로그래밍)
> **환경:** Windows / VMware Workstation 17 Player / Rocky Linux 10.2 / Bash
> **기록 범위:** 직접 수행·화면으로 검증한 **주요 실습 완료**. 교재의 세부 절 전체를 완수했다는 의미는 아님.

이 문서는 **왜 필요한지 → 어떤 구조인지 → 어떤 명령을 썼는지 → 무엇으로 성공을 검증했는지**를 기록한다. 기억이 나지 않을 때 아래 문서부터 따라가면 개념을 복습하고, _새 실습용 VM에서_ 안전하게 재현할 수 있다.

## 📚 학습 문서 목차

| 문서                                                    | 무엇을 공부하는가?                                    |
| ------------------------------------------------------- | ----------------------------------------------------- |
| [Chapter 6 상세 복습](docs/chapter06-storage.md)        | 파티션, XFS, fstab, LVM, Quota, RAID 1, 복구와 재부팅 |
| [Chapter 7 상세 복습](docs/chapter07-shell.md)          | Bash 변수·함수·조건문·반복문·인자·오류 처리·로그      |
| [실제 오류 해결 기록](docs/troubleshooting.md)          | 증상 → 원인 → 해결 → 확인, 실수로 배운 점             |
| [복습 문제와 실전 체크리스트](docs/review-questions.md) | 교수님 예상 질문, 손으로 설명하기, 재현 테스트        |

## 🎯 실습 목표

1. 디스크·파티션·파일시스템·마운트의 관계를 설명한다.
2. LVM으로 저장공간을 관리하고 데이터 유지 여부를 확인하면서 용량을 늘린다.
3. XFS Quota로 사용자별 용량 초과를 재현한다.
4. RAID 1에서 디스크 하나를 장애 처리하고, 계속 읽고 쓰고, 복구한 뒤 재부팅해 검증한다.
5. Bash 스크립트로 반복 점검·백업·실패 처리·로그 기록을 자동화한다.

## 🧰 실제 실습 환경

| 구분               | 장치 / 설정                              | 설명                                    |
| ------------------ | ---------------------------------------- | --------------------------------------- |
| 운영체제           | `/dev/nvme0n1` 40GB                      | OS가 설치된 디스크.**포맷·재분할 금지** |
| 실습 디스크        | `/dev/nvme0n2` 12GB                      | 파티션 실습용 추가 디스크               |
| 일반 파티션        | `/dev/nvme0n2p1` 4GB                     | XFS →`/mnt/labdisk`                     |
| LVM 파티션         | `/dev/nvme0n2p2` 8GB                     | PV →`vg_lab` → `lv_data`                |
| LVM 논리 볼륨      | `lv_data` 5GB → 7GB                      | XFS →`/mnt/lvmdata`                     |
| RAID 구성원        | `/dev/nvme0n3`, `/dev/nvme0n4`, 각각 2GB | RAID 1`/dev/md0` → `/mnt/raid1`         |
| 실습 사용자        | `quotauser`                              | XFS 사용자별 Quota 시험                 |
| 스크립트 작업 경로 | `~/security-lab/week03/scripts`          | 직접 작성한`.sh` 파일 3개               |

**주의:** 장치 이름은 이 실습 당시 출력에 한정된다. 다른 컴퓨터나 VM에서 동일하다고 가정하지 않는다. `fdisk`, `mkfs`, `pvcreate`, `mdadm --create`는 데이터를 손상시킬 수 있다. 이 문서는 **기존 시스템에서 그대로 재실행하는 명령 모음이 아니다.** 실습 전 `lsblk -f`, `findmnt`, `pvs`, `cat /proc/mdstat`로 장치를 확인하고 스냅샷/백업을 준비한다.

## 🗺️ 전체 구조 한눈에 보기

```text
12GB 추가 디스크 /dev/nvme0n2
 ├─ p1 : 4GB → XFS → /mnt/labdisk (백업 파일 보관)
 └─ p2 : 8GB → PV → VG(vg_lab) → LV(lv_data)
                                      └─ 5GB → 7GB, XFS → /mnt/lvmdata
                                                    └─ 사용자 Quota

2GB /dev/nvme0n3 ──┐
                    ├─ RAID 1 → /dev/md0 → XFS → /mnt/raid1
2GB /dev/nvme0n4 ──┘

Bash
 ├─ disk-check.sh  : 사용률 ↔ 임계값 비교
 ├─ backup.sh      : tar.gz 생성 → 복원 → cmp 검증
 └─ ops-review.sh  : mounts / disk / raid / repeat → check.log
```

## ✅ Chapter 6 — 직접 확인한 결과

| 구분        | 수행 내용                                  | 검증 근거                                                |
| ----------- | ------------------------------------------ | -------------------------------------------------------- |
| 파티션      | 12GB 디스크를 GPT, 4GB+8GB 파티션으로 구성 | `lsblk` 출력                                             |
| XFS         | 첫 파티션을 포맷·마운트하고`test.txt` 기록 | `df -Th`, `cat`                                          |
| 영구 마운트 | UUID 기반`/etc/fstab` 설정                 | `findmnt`, 재부팅 후 마운트 확인                         |
| LVM         | `pvcreate` → `vgcreate` → `lvcreate`       | `pvs`, `vgs`, `lvs`                                      |
| 확장        | LV 5GB → 7GB, XFS 확장                     | `df -Th /mnt/lvmdata`에서 7.0G, 기존 파일 유지           |
| Quota       | `quotauser`: Soft 100M / Hard 150M         | 200MiB 쓰기 시`Disk quota exceeded`, 약 150MiB까지 기록  |
| RAID 1      | 2GB×2 디스크 미러 구성                     | `/proc/mdstat` `[2/2] [UU]`                              |
| 장애        | 구성원 한 개`--fail`                       | `[2/1] [U_]` 상태에서도 기존 파일 읽기·새 파일 쓰기 성공 |
| 복구        | 제거→재추가, 정상 재동기화 확인            | `[2/2] [UU]`, `Failed Devices: 0`                        |
| 재부팅      | mdadm 배열·fstab 설정 후 재부팅            | `/dev/md0` 자동 조립·`/mnt/raid1` 마운트·내용 보존       |

**기억할 원리:** LVM은 _공간의 유연한 할당_, Quota는 _사용자별 공간 제한_, RAID 1은 _단일 디스크 장애 대응_, 백업은 *삭제·변조 등으로부터 이전 데이터 복구*에 초점이 있다. RAID는 백업이 아니다.

## ✅ Chapter 7 — 직접 확인한 결과

| 스크립트        | 목적                                  | 검증한 실행 결과                                                                 |
| --------------- | ------------------------------------- | -------------------------------------------------------------------------------- |
| `disk-check.sh` | `/mnt/lvmdata` 사용률을 임계값과 비교 | 사용률 3%에서`1` 입력은 WARNING, `80`은 OK                                       |
| `backup.sh`     | 파일을 날짜별 tar.gz로 압축 백업      | 압축 생성·목록 조회·별도 폴더 복원·`cmp` 종료 코드 0                             |
| `ops-review.sh` | 서버 마운트/디스크/RAID 통합 검사     | `mounts`, `disk`, `raid`, `repeat` 모드 정상                                     |
| `check.log`     | 점검 결과 누적 기록                   | `>`, `>>`, `2>&1`로 출력 기록; 잘못된 인자 `abc` → 종료 코드 2 및 오류 문구 저장 |

> **코드 출처 주의:** 정확한 `.sh` 원본 파일은 이 저장소에 별도로 올려야 한다. 이 문서의 코드는 설명용 예시이고, 스크린샷만으로 원본 전체를 재구성해 ‘실제 실행 코드’로 표시하지 않았다.

## 🧪 빠른 상태 점검 (읽기 전용)

```bash
lsblk -o NAME,SIZE,FSTYPE,TYPE,MOUNTPOINTS
sudo pvs; sudo vgs; sudo lvs
findmnt /mnt/labdisk
findmnt /mnt/lvmdata
findmnt /mnt/raid1
cat /proc/mdstat
sudo mdadm --detail /dev/md0
sudo xfs_quota -x -c 'report -h -u' /mnt/lvmdata
```

`findmnt`가 아무것도 출력하지 않으면 해당 경로의 마운트가 없는 것일 수 있다. 경로/장치 상태를 확인한다. `mdadm`은 `mdadm` 설치와 해당 RAID 존재를 전제로 한다.

## 🔍 오류 해결에서 얻은 교훈

- 파티션 용량의 `4G`와 `+4G`를 혼동하면 `Value out of range`가 날 수 있다.
- Bash의 `unexpected EOF`는 오류 줄 앞에 **닫히지 않은 따옴표**가 원인일 수 있다.
- `bash -n` 통과는 **문법 검사 성공**이지 실제 동작 보증이 아니다.
- `Permission denied`는 소유자·그룹·디렉터리 탐색/쓰기 권한부터 본다.
- 백업이 생성됐다는 것만으로 복구 가능한 것은 아니다. `tar -xzf`와 `cmp`로 검증한다.
- CLI 인자는 공백으로 나뉜다. `disk80`은 `disk 80`이 아니다.

사례와 해결 방법은 [Troubleshooting](docs/troubleshooting.md)에 설명했다.

## 🔄 30~50분 복습 순서

1. **5분:** 위의 전체 구조를 보지 않고 종이에 그린다.
2. **10분:** [Chapter 6](docs/chapter06-storage.md)에서 PV/VG/LV, Quota, RAID 복구 과정을 말로 설명한다.
3. **10분:** [Chapter 7](docs/chapter07-shell.md)에서 인자 `$1`, 종료 코드 `$?`, `if/case/for/while`, 출력 리다이렉션을 설명한다.
4. **10분:** [복습 문제](docs/review-questions.md)를 답을 가리고 푼다.
5. **15분:** VM에서 **읽기 전용 점검 명령**을 직접 재현하고 결과를 해석한다.

## 🔐 공개 저장소에 올릴 때 주의

- 비밀번호, 토큰, SSH 개인키, `/etc/shadow`, 민감한 서비스 로그는 올리지 않는다.
- 스크린샷의 사용자명·내부 IP·UUID 등의 공개 필요성을 검토한다.
- `check.log`, 실제 백업 `.tar.gz`, 대용량 실습 데이터는 기본적으로 올리지 않는다.
- `.sh`는 **직접 작성해 테스트한 원본**만 올리고, 그 외 예시는 `예제`라고 명확히 표시한다.

## 📖 학습 자료와 범위

- 『이것이 리눅스다 with Rocky Linux 10』 Chapter 6·7: **학습 범위의 기준**
- 로컬 매뉴얼: `man fdisk`, `man lvm`, `man xfs_quota`, `man mdadm`, `man bash`
- Rocky Linux 공식 문서: https://docs.rockylinux.org/

추가 학습 과제: 교재 세부 목차 대조, LVM Snapshot, 정기 백업 자동화, `trap`, `ShellCheck`, 실제 stderr 검증, RAID/백업 재해 복구 시나리오.
