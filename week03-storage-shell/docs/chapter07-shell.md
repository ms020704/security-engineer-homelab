# Chapter 7 — Bash 셸 스크립트 프로그래밍

> **학습 기준:** 『이것이 리눅스다 with Rocky Linux 10』 Chapter 7
> **정확성:** 이 문서의 짧은 코드는 **복습용 예시**다.

## 1. 셸 스크립트는 왜 필요한가?

셸(Bash)은 사용자가 작성한 명령을 실행하는 명령 해석기다. 스크립트는 여러 명령을 **변수·조건·반복·함수**와 함께 파일로 묶어 반복 작업을 자동화한다. 서버에서는 디스크 사용률 확인, 백업, RAID 모니터링, 로그 저장 등에 활용할 수 있다.

```bash
#!/bin/bash
echo 'Hello Linux'
```

| 명령/구문           | 무엇을 하는가?                                        |
| ------------------- | ----------------------------------------------------- |
| `nano hello.sh`     | 텍스트 편집기 실행                                    |
| `#!/bin/bash`       | 직접 실행할 때 사용할 Bash 인터프리터 지정(Shebang)   |
| `chmod +x hello.sh` | 파일 실행 권한 추가                                   |
| `./hello.sh`        | 현재 경로의 스크립트를 직접 실행                      |
| `bash hello.sh`     | Bash를 명시해서 파일 실행(파일 실행 권한 없어도 가능) |
| `bash -n hello.sh`  | 실제 실행 없이**문법** 검사                           |
| `bash -x hello.sh`  | 실행 과정 추적(민감정보가 노출될 수 있으므로 주의)    |

**주의:** `bash -n`의 무출력은 구문 분석 성공이다. 논리적 오류·파일 누락·권한 부족까지 검사하는 것은 아니다.

## 2. 변수와 인자

```bash
TARGET='/mnt/lvmdata'
LIMIT=${1:-80}
echo "$TARGET"
echo "$LIMIT"
```

- `TARGET=...`: 변수 대입. 등호 양옆에 공백을 넣지 않는다.
- `"$TARGET"`: 공백이나 특수문자가 포함된 값을 안전하게 전달하기 위해 인용한다.
- `$0`: 스크립트의 실행 이름/경로.
- `$1`, `$2`: 첫 번째·두 번째 위치 인자.
- `$#`: 받은 인자의 개수.
- `$?`: **직전 실행한 명령**의 종료 코드. 보통 0=성공, 그 외는 실패/별도 의미.
- `${1:-80}`: 첫 번째 인자가 없거나 비어 있으면 80을 기본값으로 사용.
- `$(date +%Y%m%d)`: 명령어 출력 결과를 변수에 삽입하는 **명령어 치환**.
- `$((i + 1))`: 정수 연산인 **산술 확장**.

스크립트를 `./ops-review.sh disk 80`으로 실행하면 `$1=disk`, `$2=80`이다. `disk80`은 인자가 하나인 완전히 다른 문자열이다.

## 3. if — 조건 분기

```bash
USAGE=3
LIMIT=1
if [ "$USAGE" -ge "$LIMIT" ]; then
  echo 'WARNING'
else
  echo 'OK'
fi
```

`[ ]` 안에는 **공백**이 필요하다. `-ge`는 greater or equal(이상)이다.

| 숫자 비교     | 뜻          |
| ------------- | ----------- |
| `-eq`         | 같다        |
| `-ne`         | 다르다      |
| `-gt` / `-ge` | 초과 / 이상 |
| `-lt` / `-le` | 미만 / 이하 |

파일/디렉터리 검사: `-f` 일반 파일, `-d` 디렉터리, `-e` 존재, `-r` 읽기 가능, `-w` 쓰기 가능, `-x` 실행/탐색 가능.

숫자 비교 전에 문자열 검사가 필요하다. 예시 `[[ "$LIMIT" =~ ^[0-9]+$ ]]`는 숫자로만 이루어졌는지 확인한다. 실제 `ops-review.sh`는 `abc`를 잘못된 값으로 처리했고 종료 코드 2를 반환했다.

## 4. case — 여러 동작 선택

```bash
ACTION=${1:-}
case "$ACTION" in
  mounts) echo 'Mounts' ;;
  disk)   echo 'Disk' ;;
  raid)   echo 'RAID' ;;
  *)      echo 'Usage error'; exit 2 ;;
esac
```

`in`이 패턴 목록을 시작하고 `패턴)`이 매칭 대상을 표시한다. `;;`는 분기를 끝내고 `*)`는 나머지 모든 값, `esac`는 case 종료다. 실제 `ops-review.sh`에는 `repeat` 분기도 있었다.

## 5. for / while — 반복 실행

```bash
for DIR in /mnt/labdisk /mnt/lvmdata /mnt/raid1; do
  if mountpoint -q "$DIR"; then
    echo "[OK] $DIR"
  fi
done
```

`for`는 정해진 목록을 순회할 때 편리하다. `mountpoint -q`는 출력 없이 종료 코드로 마운트 여부를 알려준다.

```bash
i=1
while [ "$i" -le 3 ]; do
  echo "Check number: $i"
  i=$((i + 1))
done
```

`while`은 조건이 참인 동안 반복한다. 증가 부분이 없으면 무한 반복할 수 있다. 실습에서 `repeat`라는 **문자열을 정수로 비교**해 `integer expression expected` 오류가 났고, 반복 변수 `i`를 사용해 해결했다.

## 6. 함수 — 반복 코드를 하나로

```bash
check_mount() {
  local target=$1
  mountpoint -q "$target"
}
if check_mount /mnt/raid1; then
  echo '[OK] mounted'
fi
```

함수는 특정 동작을 재사용하게 한다. `local`은 함수의 지역 변수 선언, `return 1`은 함수 실행 상태 반환, `exit 1`은 **스크립트 자체** 종료이다.

## 7. 파이프와 로그 저장

| 구문                  | 용도                                                 |
| --------------------- | ---------------------------------------------------- |
| `cmd > out.log`       | 표준 출력 덮어쓰기                                   |
| `cmd >> out.log`      | 표준 출력 추가                                       |
| `cmd 2> err.log`      | 표준 오류만 파일 기록                                |
| `cmd >> all.log 2>&1` | 표준 출력 추가·표준 오류도 같은 대상에 연결          |
| `cmd1 \| cmd2`        | 앞 명령의 표준 출력을 다음 명령의 표준 입력으로 전달 |

순서는 중요하다. `> out.log 2>&1`과 `2>&1 > out.log`는 같은 뜻이 아니다.

실제로 검증한 명령:

```bash
./ops-review.sh mounts > ~/security-lab/week03/check.log 2>&1
./ops-review.sh raid >> ~/security-lab/week03/check.log 2>&1
./ops-review.sh disk abc >> ~/security-lab/week03/check.log 2>&1
echo $?
tail -n 10 ~/security-lab/week03/check.log
```

확인: 마운트 3개 정상, RAID `[2/2] [UU]`, `ERROR: Limit must be 1-100`, 종료 코드 `2`. 단, 이 문구는 `echo`를 통해 **표준 출력**으로 출력했을 수 있으므로 `2>&1`을 이용한 **실제 stderr 출력 테스트**와는 구별한다.

추가 연습:

```bash
bash -c 'echo "stderr check" >&2' >> ~/security-lab/week03/check.log 2>&1
tail -n 2 ~/security-lab/week03/check.log
```

## 8. 만든 스크립트 3개 이해하기

### `disk-check.sh` — 디스크 사용률

**무엇을?** `/mnt/lvmdata`가 마운트되어 있는지 확인하고 현재 사용률을 인자로 전달한 기준과 비교한다.

**어떻게?** `mountpoint` → `df -P` → `awk`로 `%` 제거 → `[ "$USAGE" -ge "$LIMIT" ]` → 경고 또는 OK.

| 실행 조건           | 관찰한 결과                    |
| ------------------- | ------------------------------ |
| 사용률 3%, 기준`1`  | `WARNING: Disk usage is high!` |
| 사용률 3%, 기준`80` | `OK: Disk usage is normal.`    |

문법 오류 `unary operator expected`를 수정하고 비교 결과를 재확인했다.

### `backup.sh` — 압축 백업

**무엇을?** `/mnt/lvmdata/test.txt`를 `/mnt/labdisk/backups/backup_날짜.tar.gz` 형태로 보관.

**어떻게?** 마운트/원본 확인 → 출력 디렉터리 → 파일명 시각 부여 → `tar -czf` → `tar -tzf` → `tar -xzf` → `cmp`.

| 명령              | 의미                                  |
| ----------------- | ------------------------------------- |
| `tar -czf`        | 새 gzip 압축 아카이브 생성            |
| `tar -tzf`        | 압축 내부 파일 목록 조회              |
| `tar -xzf`        | 압축 풀기                             |
| `tar -C 경로`     | 지정 경로를 기준으로 파일 선택/해제   |
| `cmp 원본 복원본` | 바이트 단위 비교, 일치 시 종료 코드 0 |

실제 결과: 압축 파일 생성, 잘못된 경로 수정, 복원, `cmp` 성공.

### `ops-review.sh` — 운영 점검

```bash
./ops-review.sh mounts   # /mnt/labdisk /mnt/lvmdata /mnt/raid1 점검
./ops-review.sh disk 1   # 3% 사용률 -> WARNING
./ops-review.sh disk 80  # 3% 사용률 -> OK
./ops-review.sh raid     # /proc/mdstat 표시
./ops-review.sh repeat   # 마운트 점검 3회 반복
./ops-review.sh unknown  # 사용법 표시/종료 코드 2
```

`case` 분기, `for`/`while`, 위치 인자와 조건문을 조합했다. 직접 테스트한 값 `disk abc`는 오류 문구와 종료 코드 2를 냈다.

## 9. 디버깅에서 기억할 내용

| 증상                                      | 무엇부터 검사?                                          |
| ----------------------------------------- | ------------------------------------------------------- |
| `unexpected EOF`                          | 닫지 않은 작은/큰따옴표, 괄호,`fi`/`done`               |
| `cannot execute: required file not found` | Shebang 경로(실습 중`#!/bin/bah` 오타), 줄바꿈 문자     |
| `unexpected token ')'`                    | `case "$ACTION" in`과 각 `패턴)`의 문법                 |
| `integer expression expected`             | 정수 비교에 문자열이 들어갔는지                         |
| `Permission denied`                       | 파일 실행 비트뿐 아니라 대상**디렉터리 쓰기·탐색** 권한 |
| `bash -n` 정상, 실행 실패                 | 문법 외 권한·파일 누락·조건/변수 값 확인                |

`set -euo pipefail`의 의미: `-e`는 일부 예외를 제외하고 실패 시 종료, `-u`는 미정의 변수 오류, `pipefail`은 파이프 중간 실패도 반영. 단, 이 설정만으로 모든 예외 처리가 해결되는 것은 아니다.
