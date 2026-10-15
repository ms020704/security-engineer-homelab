# Chapter 6 — 디스크 관리·사용자별 공간 할당

> **학습 기준:** 『이것이 리눅스다 with Rocky Linux 10』 Chapter 6
> **재현 원칙:** 아래 `fdisk`/`mkfs`/`pvcreate`/`mdadm --create`는 **빈 실습용 디스크에서만** 사용한다. 기존 VM에 재실행하지 않는다. 장치명은 시스템마다 다르다.

## 1. 개념을 먼저 연결하기

```text
Disk → Partition → Filesystem → Mount → File
Disk → Partition → PV → VG → LV → Filesystem → Mount → File
2 disks → RAID 1(/dev/md0) → Filesystem → Mount
```

| 개념            | 정의                                  | 왜 필요한가?                    |
| --------------- | ------------------------------------- | ------------------------------- |
| 디스크          | 물리/가상 블록 저장장치               | 데이터 저장의 바탕              |
| 파티션          | 디스크의 구획                         | 목적별 저장 영역 분리           |
| 파일시스템(XFS) | 파일·디렉터리 구조와 메타데이터 관리  | 공간을 실제 파일로 사용         |
| 마운트          | 파일시스템을 Linux 경로에 연결        | 파일 경로로 접근                |
| PV              | LVM 물리 볼륨                         | LVM 관리 대상으로 등록          |
| VG              | 물리 볼륨의 공간 풀                   | LV에 할당할 공간 확보           |
| LV              | VG에서 만든 논리 볼륨                 | 파일시스템이 놓일 논리 블록장치 |
| UUID            | 파일시스템/배열 등 객체를 식별하는 값 | 장치명이 바뀌어도 식별 가능     |

### 기본 확인 명령

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS # 저장장치 계층 관계
lsblk -f                                # 파일시스템/UUID/마운트
sudo blkid                              # 블록장치의 식별정보
findmnt /mnt/labdisk                    # 경로별 마운트 원천
df -Th /mnt/labdisk                     # 사용량/종류/마운트
sudo pvs; sudo vgs; sudo lvs            # LVM의 3계층 현황
```

**설명 테스트:** `lsblk`는 디스크 구조에 강하고, `df`는 *이미 마운트된 파일시스템*의 사용량 확인에 강하다. `lsblk`의 파티션 8GB가 모두 사용 가능한 XFS 공간이라는 뜻은 아니다.

## 2. 12GB 디스크를 4GB+8GB로 분할 (실제 수행 과정 기록)

- 디스크: `/dev/nvme0n2` **12GB**, 운영체제는 별도 `/dev/nvme0n1` **40GB**.
- `/dev/nvme0n2p1` 4GB: XFS, `/mnt/labdisk`.
- `/dev/nvme0n2p2` 8GB: LVM 타입.

실습에서 `sudo fdisk /dev/nvme0n2`로 진입해 사용한 대화형 명령:

| 키    | 의미                                                                |
| ----- | ------------------------------------------------------------------- |
| `g`   | 새 GPT 파티션 테이블 생성 (**기존 구조 파괴 가능**)                 |
| `n`   | 새 파티션                                                           |
| Enter | 기본 파티션 번호·시작 섹터 등 선택                                  |
| `+4G` | 시작 섹터로부터 4GiB 할당                                           |
| `t`   | 파티션 타입 변경 (두 번째 파티션을 Linux LVM으로)                   |
| `44`  | 해당 환경에서 Linux LVM에 매핑된 타입 번호(다른 버전에선`L`로 확인) |
| `p`   | 변경 예상 구조 조회                                                 |
| `w`   | 디스크에 기록 (주의)                                                |
| `q`   | 기록하지 않고 종료                                                  |

**실수와 수정:** 마지막 섹터 입력에 `4G` → `Value out of range`, 다시 `+4G`로 입력하니 4GiB 생성에 성공했다. `4G`는 그 위치에서 유효한 마지막 섹터 지정이 아니며 `+4G`는 상대 크기 지정이다.

> 재현하려면 **새로 추가한 빈 디스크를 `lsblk`로 식별한 뒤** 진행한다. 기존 `/dev/nvme0n2`에 다시 `g`나 `w`를 실행하면 현재 자료가 손실될 수 있다.

## 3. XFS 포맷·마운트와 /etc/fstab

실습 당시 최초 구성에 사용한 명령(기존 파티션에 재실행 금지):

```bash
sudo mkfs.xfs /dev/nvme0n2p1
sudo mkdir -p /mnt/labdisk
sudo mount /dev/nvme0n2p1 /mnt/labdisk
findmnt /mnt/labdisk
df -Th /mnt/labdisk
echo 'Chapter 6 Disk Test' | sudo tee /mnt/labdisk/test.txt
cat /mnt/labdisk/test.txt
```

- `mkfs.xfs`: 파티션에 XFS 파일시스템 생성 (기존 파일시스템 손상 가능).
- `mkdir -p`: 마운트 지점 디렉터리 생성; 이미 존재해도 오류 내지 않음.
- `mount DEVICE PATH`: 저장장치의 파일시스템을 해당 경로에 연결.
- `tee`: 표준 입력을 파일에 **기록하면서 화면에도 출력**; `sudo echo ... > 파일`의 권한 문제를 피하는 용도로 사용 가능.

### fstab으로 재부팅 후 자동 마운트

```bash
sudo blkid /dev/nvme0n2p1
sudo cp -a /etc/fstab /etc/fstab.before-week03
sudo nano /etc/fstab
```

입력 형식 예시(**UUID를 조회한 실습값으로 대체**):

```fstab
UUID=<p1에서 확인한-UUID> /mnt/labdisk xfs defaults,nofail 0 0
```

| 항목              | 의미                                |
| ----------------- | ----------------------------------- |
| UUID=...          | 파일시스템 식별자                   |
| `/mnt/labdisk`    | 마운트 지점                         |
| `xfs`             | 파일시스템 종류                     |
| `defaults,nofail` | 기본 옵션·마운트 실패해도 부팅 계속 |
| `0 0`             | dump/부팅 시 fsck 검사 순서 설정    |

```bash
sudo systemctl daemon-reload
sudo findmnt --verify --verbose
findmnt /mnt/labdisk
```

`findmnt --verify`는 설정 검증에 유용하다. 부팅 가능 여부와 실제 데이터 유지까지 보장하는 것은 아니므로 재부팅 시험 시 `findmnt`, `cat`으로 재확인한다. `nofail`로 인해 디스크가 없더라도 부팅할 수 있으므로 **부팅 성공 = 마운트 성공**은 아니다.

## 4. LVM: PV → VG → LV

**이유:** 파일시스템 용량을 운영 중 확장하려면 장치 관리 계층이 중요하다. LVM은 PV를 VG로 묶고 VG 공간에서 LV를 배정하여 유연하게 관리한다.

실습 최초 구성 시 사용한 명령(**이미 구축한 환경에서 다시 실행하지 않음**):

```bash
sudo pvcreate /dev/nvme0n2p2                 # 파티션을 LVM PV로 초기화
sudo vgcreate vg_lab /dev/nvme0n2p2           # VG 생성
sudo lvcreate -L 5G -n lv_data vg_lab         # VG에서 5GiB LV 생성
sudo mkfs.xfs /dev/vg_lab/lv_data             # LV에 XFS 생성
sudo mkdir -p /mnt/lvmdata
sudo mount /dev/vg_lab/lv_data /mnt/lvmdata
sudo pvs; sudo vgs; sudo lvs
df -Th /mnt/lvmdata
```

옵션 해설:

- `-L 5G`: LV의 **절대 크기**를 5GiB로 지정.
- `-n lv_data`: 논리 볼륨 이름 지정.
- `vg_lab`: 할당할 공간을 가진 VG 이름.

### 5GiB → 7GiB 확장

```bash
sudo lvextend -L 7G /dev/vg_lab/lv_data     # LV 자체의 크기 증가
sudo lvs                                  # LV 크기 확인
df -Th /mnt/lvmdata                       # 파일시스템은 여전히 이전 크기일 수 있음
sudo xfs_growfs /mnt/lvmdata               # 마운트된 XFS 파일시스템 확장
df -Th /mnt/lvmdata                       # 최종 약 7.0G 검증
cat /mnt/lvmdata/test.txt                  # 기존 내용 유지 검증
```

**핵심 답변:** `lvextend`는 블록장치의 논리 볼륨 크기를, `xfs_growfs`는 **마운트된 XFS 파일시스템의 사용 가능 크기**를 확장한다. 둘은 별도 계층이다. XFS는 이 경우 온라인 확장이 가능하다. `-L 7G`는 **최종 크기** 7GiB이고, `-L +2G`는 기존 크기에서 2GiB 추가한다.

실제 결과: `df -Th /mnt/lvmdata`에서 5.0G → 7.0G, `test.txt` 원본 유지.

## 5. XFS 사용자 Quota

**왜?** 공유 서버에서 사용자 한 명이 파일시스템 공간을 독점하지 못하도록 사용량을 제한한다.

- Soft Limit: 초과 시 **유예기간**이 적용될 수 있는 기준.
- Hard Limit: 초과해서 쓸 수 없는 상한.
- Quota에는 사용량 계수(Accounting)와 제한 강제(Enforcement)가 필요.

실습에서는 `/mnt/lvmdata`에 사용자 Quota를 설정하고 `xfs_quota`의 `state` 출력에서 **User Accounting ON, Enforcement ON**을 확인했다. 사용된 fstab 설정에는 `uquota`가 포함됐다.

```bash
sudo xfs_quota -x -c 'state' /mnt/lvmdata
sudo useradd -m quotauser   # 새 실습 VM에서 계정이 없을 때만
sudo mkdir -p /mnt/lvmdata/quotauser
sudo chown quotauser:quotauser /mnt/lvmdata/quotauser
sudo chmod 700 /mnt/lvmdata/quotauser
sudo xfs_quota -x -c 'limit -u bsoft=100m bhard=150m quotauser' /mnt/lvmdata
sudo xfs_quota -x -c 'report -h -u' /mnt/lvmdata
```

| 인자           | 의미                               |
| -------------- | ---------------------------------- |
| `-x`           | XFS quota 관리자 모드              |
| `-c '...'`     | 지정한 하위 명령 실행              |
| `limit -u`     | 사용자별 블록 사용량 제한          |
| `bsoft=100m`   | Soft Limit 100MiB 수준             |
| `bhard=150m`   | Hard Limit 150MiB 수준             |
| `report -h -u` | 사용자별 현재 사용량과 제한값 표시 |

### 실패 상황을 의도적으로 재현

```bash
sudo -u quotauser dd if=/dev/zero of=/mnt/lvmdata/quotauser/test.bin bs=1M count=200 status=progress
sudo xfs_quota -x -c 'report -h -u' /mnt/lvmdata
sudo du -h /mnt/lvmdata/quotauser/test.bin
```

- `sudo -u quotauser`: 지정 사용자 권한으로 명령 실행.
- `dd`: 지정 블록 단위로 데이터 기록.
- `if=/dev/zero`: 0이 반복되는 입력 데이터.
- `of=`: 출력 대상 파일.
- `bs=1M`, `count=200`: 1MiB×200회 = 200MiB 쓰기 시도.

실제 결과: `Disk quota exceeded`, 파일 사용량 약 150M, 보고서에 Soft 100M/Hard 150M·유예기간 표시. **전체 디스크 여유 공간과 관계없이 사용자별 할당량 때문에 실패**했다.

정리: `sudo -u quotauser rm /mnt/lvmdata/quotauser/test.bin` → report에서 사용량이 줄었는지 확인했다.

## 6. RAID 1 구성·고장·복구

**RAID 1 원리:** 두 디스크에 같은 데이터를 기록해 구성원 한 개가 실패해도 접근을 이어갈 수 있는 **미러링**. 두 개의 2GB 디스크를 썼을 때 사용 가능한 공간은 약 한 개 디스크 용량이다(메타데이터 제외).

RAID/백업 구분: RAID는 디스크 장애 시 **가용성**을 보조하지만 파일 실수 삭제/랜섬웨어/논리적 손상은 양쪽에 반영될 수 있다. 독립적으로 보관되는 복원 가능한 백업이 별도로 필요하다.

### 새 빈 디스크에서 처음 구성할 때 사용한 과정 (파괴적)

```bash
sudo dnf install -y mdadm
sudo mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/nvme0n3 /dev/nvme0n4
cat /proc/mdstat
sudo mdadm --detail /dev/md0
sudo mkfs.xfs /dev/md0
sudo mkdir -p /mnt/raid1
sudo mount /dev/md0 /mnt/raid1
```

**절대 기존 배열에 `mdadm --create` 또는 `mkfs.xfs`를 다시 실행하지 않는다.**

```bash
echo 'RAID 1 Data Protection Test' | sudo tee /mnt/raid1/test.txt
cat /mnt/raid1/test.txt
sync
```

- `/proc/mdstat`: 커널이 관리하는 MD RAID 상태.
- `mdadm --detail`: RAID 수준·구성원·장애 장치 등 자세한 정보.
- `[2/2] [UU]`: 두 구성원이 모두 사용 가능한 정상 상태.

### 의도적 고장 시나리오 — 실습용 RAID에서만

```bash
sudo mdadm /dev/md0 --fail /dev/nvme0n4
cat /proc/mdstat
sudo mdadm --detail /dev/md0
cat /mnt/raid1/test.txt
echo 'RAID 1 still working' | sudo tee /mnt/raid1/after-failure.txt
cat /mnt/raid1/after-failure.txt
```

실제 확인: `[2/1] [U_]`, `clean, degraded`, 실패 디스크 1개. **이 상태에서 기존 파일 읽기·신규 파일 쓰기 성공**.

### 복구

```bash
sudo mdadm --manage /dev/md0 --remove /dev/nvme0n4
sudo mdadm --manage /dev/md0 --add /dev/nvme0n4
cat /proc/mdstat
sudo mdadm --detail /dev/md0
```

`--remove`는 **이미 실패 처리된 구성원**을 제거한다. 실제 디스크 장애라면 고장난 물리 디스크를 교체한 후 새 장치가 무엇인지 확인해야 한다. 실습에서는 같은 가상 디스크를 다시 추가했다.

결과: `[2/2] [UU]`, `State: clean`, `Active Devices: 2`, `Failed Devices: 0`. 기존/장애 중 생성한 파일 모두 유지.

### RAID 자동 인식 + 자동 마운트

```bash
sudo mdadm --detail --scan
sudo mdadm --detail --scan | sudo tee /etc/mdadm.conf
sudo blkid /dev/md0
sudo cp -a /etc/fstab /etc/fstab.before-raid1
sudo nano /etc/fstab
```

이때 `/etc/mdadm.conf`에 넣는 **MD 배열 UUID**와, `/etc/fstab`에 넣는 **XFS 파일시스템 UUID**는 다른 값이다. 사진에서 파일시스템 UUID는 `e86d96b1-06ae-49c4-8594-609275c047a7`이었다. 마운트 예시:

```fstab
UUID=<blkid /dev/md0에서-확인한-파일시스템-UUID> /mnt/raid1 xfs defaults,nofail 0 0
```

재부팅 전에 `/etc/fstab` 오타 검사, RAID 상세정보 점검, 별도 데이터 백업을 진행한다. 실제 실행 후 재부팅을 통해 검증했다.

```bash
cat /proc/mdstat
sudo mdadm --detail /dev/md0
findmnt /mnt/raid1
cat /mnt/raid1/test.txt
cat /mnt/raid1/after-failure.txt
```

재부팅 후 `[UU]`, `Failed Devices: 0`, 마운트된 `/dev/md0`, 파일 내용 유지가 화면으로 확인됐다. **부팅했다는 사실보다 실제 배열·마운트·파일 내용 확인이 핵심이다.**

## 7. 비교·설명 암기표

| 비교                       | 가장 중요한 차이                             |
| -------------------------- | -------------------------------------------- |
| `lsblk` vs `df`            | 장치 계층 구조 vs 마운트된 파일시스템 사용량 |
| VG vs LV                   | 공간 풀 vs 풀에서 배정한 실제 논리 볼륨      |
| `lvextend` vs `xfs_growfs` | 블록 LV 크기 확장 vs XFS 자체 확장           |
| Quota vs 디스크 꽉 참      | 사용자별 제한 vs 파일시스템 전체 사용량      |
| Soft vs Hard               | 유예가능 경계 vs 쓰기 금지 상한              |
| `[UU]` vs `[U_]`           | 2/2 정상 vs 1/2 degraded                     |
| mdadm UUID vs XFS UUID     | 배열 자체 구분 vs 내부 파일시스템 구분       |
| RAID 1 vs Backup           | 장애 시 이용 가능성 vs 이전 시점 데이터 복구 |
