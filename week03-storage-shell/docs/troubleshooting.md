# Week03 — 실제 실습 오류·원인·해결 기록

> 서버 운영에서 중요한 것은 무오류가 아니라 **현상 → 원인 추정 → 확인 → 수정 → 재시험**이다. 직접 겪은 오류를 잊지 않도록 정리했다.

| 상황                 | 관찰한 메시지/증상                              | 원인                                        | 조치 및 검증                                                  |
| -------------------- | ----------------------------------------------- | ------------------------------------------- | ------------------------------------------------------------- |
| 파티션 크기 입력     | `Value out of range`                            | 마지막 섹터 칸에`4G` 입력                   | 상대 크기인`+4G` 입력 → 4GiB 파티션 생성                      |
| fstab 수정           | systemd가 예전 설정 사용 경고                   | `/etc/fstab`만 변경해 systemd가 미갱신      | `sudo systemctl daemon-reload` 이후 재확인                    |
| Quota 시험           | `Disk quota exceeded`                           | Hard Limit 150M 초과 시도(의도된 실패)      | 약 150M 기록·Quota 리포트 확인, 파일 삭제 후 사용량 0 확인    |
| 디스크 점검 스크립트 | `[: 3: unary operator expected`                 | `[` 비교식의 잘못된 공백/피연산자 처리      | 조건식 수정 후 1%에서는 WARNING·80%에서는 OK 확인             |
| 백업 문법            | `unexpected EOF while looking for matching ...` | 앞줄의 닫히지 않은 따옴표                   | 따옴표·변수 구문 수정,`bash -n` 통과                          |
| 실행 자체 불가       | `cannot execute: required file not found`       | Shebang이`#!/bin/bah`로 잘못 입력됨         | `#!/bin/bash`로 변경해 실행 성공                              |
| 백업 디렉터리        | `Permission denied`                             | 사용자에게 생성 권한 없음                   | `/mnt/labdisk/backups`만 생성 후 소유자/권한 수정 → 백업 완료 |
| tar 확인             | `No such file or directory`                     | `backup_backup_...`처럼 파일명을 중복 입력  | `ls -lh`로 실제 파일명을 확인해 수정 → tar 목록 조회 성공     |
| 복원 비교            | `cmp: ... No such file`                         | `restore-test/test/txt` 경로 입력 실수      | `restore-test/test.txt`로 수정 → `cmp`, `echo $?` = 0         |
| case 문법            | `unexpected token ')'`                          | `case ... in` 부분 문법 오류                | 구조 수정,`bash -n` 통과, 하위 명령 실행 성공                 |
| repeat 실행          | `integer expression expected`                   | 숫자 변수 대신`repeat` 문자열을 정수로 비교 | 반복 횟수 변수 수정 → 마운트 점검 세 차례 수행                |
| CLI 인자             | `disk1`, `disk80`에 Usage 출력                  | 인자 사이 공백 없음                         | `disk 1`, `disk 80`으로 실행                                  |
| RAID 장애            | `[2/1] [U_]`/`degraded`                         | 의도적으로 한 RAID 멤버를 실패 처리         | 장애 중 파일 읽기·쓰기 → 실패 멤버 remove/add →`[UU]`         |
| 잘못된 임계값        | `ERROR: Limit must be 1-100`                    | `disk abc` 입력                             | 종료 코드 2와`check.log` 기록 확인(정상적인 오류 처리)        |

## 공통 점검 루틴

1. 오류 메시지를 그대로 기록하고, 실행한 명령과 현재 경로를 확인한다.
2. 부작용 없는 확인부터: `pwd`, `ls -la`, `lsblk -f`, `findmnt`, `cat`, `bash -n`.
3. 오타 → 인자/따옴표 → 권한 → 장치·마운트 → 시스템 로그 순으로 범위를 좁힌다.
4. 수정은 최소한으로 하고, **똑같은 입력을 다시 실행해** 결과를 비교한다.
5. 성공 기준을 출력·종료 상태·데이터 유지로 명확히 남긴다.

### 예제: Bash 오류 재현 시 점검

```bash
bash -n ./ops-review.sh
bash -x ./ops-review.sh mounts
```

`bash -x`는 실제 명령/변수의 일부를 노출할 수 있으므로 보안정보가 있는 운영 환경에서는 로그 저장·공개를 주의한다.

### 예제: 복원 결과 검증

```bash
cmp /mnt/lvmdata/test.txt ~/security-lab/week03/restore-test/test.txt
echo $?
```

이 테스트에서 `0`은 파일 내용이 같았다는 뜻이다. 아무것도 출력되지 않는다는 이유만으로 성공이라고 단정하지 않는다.

### 위험한 복구 시도 금지

`mkfs.xfs`, `fdisk`에서 `w`, `mdadm --create`, `pvcreate`는 기존 데이터를 손상시킬 수 있다. 장치가 없거나 파일이 보이지 않는다고 **다시 포맷하지 않는다.** `lsblk -f`, `pvs/vgs/lvs`, `cat /proc/mdstat`로 현재 상태부터 확인한다.

### 학습 당시 검증하지 않은 것

`2>&1`을 사용해 잘못된 임계값의 **문구를 로그에 저장**한 것까지 확인했다. 해당 오류 문구가 Bash의 진짜 표준 오류(fd 2)에서 나왔는지는 화면만으로 검증되지 않았다. 실제 stderr 테스트는 추후 과제다.
