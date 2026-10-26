# Week04 — 장기 복습 질문 35제

> 사용법: **정답을 보기 전에 구두로 20~40초 설명**한다. 정의만 외우지 말고, 이번 실습에서 확인한 명령과 결과를 함께 말해본다. 문제와 정답을 분리해 놓았다.

[메인 README](../README.md) · [Chapter 8 해설](chapter08-remote-access.md) · [Chapter 9 해설](chapter09-dns.md)

## 1. 문제 먼저 풀기

### Chapter 8 — 원격지 관리

1. SSH를 사용하는 이유를 Telnet과 비교해 설명해라.
2. SSH 서버와 SSH 클라이언트의 차이는 무엇인가? 포트는?
3. `systemctl is-active sshd`와 `systemctl is-enabled sshd`는 무엇이 다른가?
4. `sudo ss -tlnp | grep ':22'`의 옵션을 풀어 설명해라.
5. `sudo sshd -T`는 무엇을 확인하며 설정 파일 읽기와 어떤 차이가 있는가?
6. SSH 연결에서 서버 **호스트 키**는 누구를 검증하는가?
7. SSH **사용자 공개키/개인키**는 누구를 인증하고 각각 어디에 보관하는가?
8. `authorized_keys`와 `known_hosts`는 어떻게 다른가?
9. SSH 공개키 인증에서 개인키를 서버에 보내지 않아도 되는 이유는?
10. `Enter passphrase for key`가 나타났다면 Linux 계정 비밀번호를 입력한 것인가?
11. `PreferredAuthentications=publickey`와 `PasswordAuthentication=no`를 함께 쓴 이유는?
12. `scp`는 무엇인가? 실제로 어떻게 전송 성공을 증명했는가?
13. 화면에서 `permitrootlogin yes`와 `passwordauthentication yes`가 나왔다. 무엇을 시사하는가?
14. SSH 설정을 수정할 때 원격 접속이 끊기는 사고를 어떻게 예방하는가?
15. XRDP와 GNOME Remote Desktop이 같은 것인가? 이번에는 실제 GUI 접속을 했는가?

### Chapter 9 — DNS

16. DNS는 단순히 이름을 IP로 바꾸는 시스템인가?
17. 권한 네임 서버와 재귀 리졸버의 차이를 설명해라.
18. 일반 인터넷 DNS 조회에서 Root·TLD·권한 서버의 역할은?
19. 이번 실습에서 Root/TLD까지 질의하지 않고 답을 받을 수 있었던 이유는?
20. A, AAAA, PTR의 차이는 무엇인가?
21. NS, SOA, CNAME, MX, TXT는 각각 어떤 정보를 제공하는가?
22. 정방향 Zone `lab.test`에서 `web IN A 192.168.199.132`는 무엇을 의미하는가?
23. `199.168.192.in-addr.arpa`가 역방향 Zone인 이유는? `132 IN PTR web.lab.test.`는?
24. DNS 이름 맨 뒤의 점(`.`)과 `@`는 무엇을 의미하는가?
25. Zone의 SOA Serial을 변경하면 어떤 의미가 있는가?
26. `TTL 3600`의 장점과 단점은 무엇인가?
27. `named.conf`와 Zone 파일의 역할 차이는?
28. `named-checkconf`·`named-checkzone`·`dig` 각각 무엇을 검증하는가?
29. DNS가 사용하는 TCP/UDP 포트는? 왜 둘 다 필요한가?
30. Linux에서 `dig @127.0.0.1`과 `dig @192.168.199.132`는 무엇이 다른가?
31. Windows에서 `Resolve-DnsName -Server 192.168.199.132`가 성공했다. 기본 DNS가 변경된 것인가?
32. `allow-query`, `allow-transfer`, `allow-update`, `recursion no`는 어떤 보안 정책인가?
33. Linux에서 DNS 질의는 성공하지만 Windows에서 실패한다면 무엇부터 점검할까?
34. `NXDOMAIN`, `REFUSED`, `SERVFAIL`, timeout은 각각 무엇을 시사하는가?
35. 이번 Week04에서 **실제로 완료한 것**과 **남은 것**을 구분해 설명해라.

---

## 2. 정답 및 해설

**1.** SSH는 암호화된 원격 채널을 통해 데이터 전송을 보호하고 서버 검증·사용자 인증을 제공한다. Telnet 원격 로그인은 기본적으로 이러한 암호화를 제공하지 않는다. 기본 포트는 SSH TCP 22, Telnet TCP 23.

**2.** 클라이언트는 연결을 시작하는 Windows의 `ssh`, 서버는 연결 요청을 받는 Linux의 `sshd`다. SSH는 일반적으로 TCP 22에서 수신한다.

**3.** `is-active`는 지금 실행 중인지 확인하고, `is-enabled`는 부팅 시 실행되도록 설정되었는지 확인한다. 이번에는 모두 `active`·`enabled`였다.

**4.** `ss`는 소켓 조회. `-t` TCP, `-l` LISTEN, `-n` 숫자 출력, `-p` 프로세스, grep은 `:22` 줄 추출. 실습에서 `0.0.0.0:22`, `[::]:22`와 `sshd`를 확인했다.

**5.** `sshd -T`는 데몬 설정의 실효 값을 출력한다. 여러 설정 파일/기본값 등이 반영될 수 있다. 특정 접속의 Match 문맥을 정확히 보려면 `-C` 조건도 고려해야 한다.

**6.** 서버 호스트 키는 **서버 자신**의 신원을 확인한다. 클라이언트가 신뢰한 호스트 키는 `known_hosts`에 기록된다.

**7.** 사용자 개인키는 Windows 클라이언트에 보관하고 공개키는 Linux 사용자의 `authorized_keys`에 등록한다. 서버는 해당 키로 인증하는 사용자를 검증한다.

**8.** `authorized_keys`는 **이 계정에 들어올 수 있는 사용자 공개키 목록**, `known_hosts`는 **클라이언트가 만난 SSH 서버의 호스트 키 목록**이다.

**9.** 클라이언트는 개인키로 서명을 생성하고 서버는 이미 등록한 공개키로 서명을 검증한다. 개인키 원문을 서버로 보내지 않아도 된다.

**10.** 아니다. 키를 잠금 해제하는 로컬 **passphrase**다. Linux 계정 비밀번호와 구별해야 한다.

**11.** 실제 공개키 인증이 가능한지 확인하기 위해 사용했다. 서버 비밀번호 인증으로 조용히 폴백하는 상황을 방지한다.

**12.** `scp`는 SSH를 활용한 파일 복사 명령이다. Windows에서 100% 전송 후 Rocky의 `cat ~/week04.txt` 결과가 `SSH file transfer test`로 나오는 것을 확인했다.

**13.** 현재 설정에서 root SSH 로그인 및 비밀번호 인증이 허용될 가능성이 있음을 뜻한다. 공개키 인증 성공과 별개로, 실제 서버에서는 더 제한적인 정책과 위협 모델을 검토해야 한다. 이번에는 **변경을 적용하지 않았다**.

**14.** VM 콘솔을 확보하고, 설정 백업→문법 검사 `sshd -t`→기존 세션 유지→새 SSH 세션으로 인증 테스트→로그 확인 순서로 안전하게 변경한다.

**15.** 같다 볼 수 없다. 둘 다 RDP 연결과 관련 있지만 서로 다른 구현이다. Rocky Linux 10의 GNOME Remote Desktop은 별도 기능이며 **이번 실제 GUI 연결은 미검증**이다.

**16.** 아니다. 이름/IP뿐 아니라 메일 서버(MX), Zone 위임(NS), TXT 같은 다양한 데이터를 계층적·분산적으로 관리한다.

**17.** 권한 서버는 자신이 관리하는 Zone의 원본 DNS 레코드에 답한다. 재귀 리졸버는 다른 서버들을 질의하고 캐시해 클라이언트 대신 최종 답을 찾는다.

**18.** Root는 해당 TLD 서버의 위치를 알려주고, TLD는 도메인 권한 서버를 안내하며, 권한 서버가 해당 Zone의 레코드를 제공한다. 중간에 캐시가 있으면 일부 단계를 건너뛸 수 있다.

**19.** Rocky의 `named`가 `lab.test` Zone을 직접 가지고 있고 Windows가 `-Server 192.168.199.132`로 그 서버를 지정했기 때문이다.

**20.** A는 이름→IPv4, AAAA는 이름→IPv6, PTR은 역방향 이름 정보를 제공한다. PTR은 신원 인증 수단이 아니다.

**21.** NS는 권한 DNS 서버, SOA는 Zone 관리 메타데이터, CNAME은 별칭 대상 이름, MX는 메일 처리 서버, TXT는 텍스트 정보다.

**22.** `web`은 Zone 기점 `lab.test`와 결합되어 `web.lab.test.`가 된다. 그 이름의 A 레코드를 `192.168.199.132`로 지정했다.

**23.** IPv4 역조회 이름은 주소의 옥텟 순서를 뒤집어 `in-addr.arpa`에 등록한다. `/24` Zone의 마지막 호스트 번호 `132`에 PTR로 `web.lab.test.`를 기록했다.

**24.** DNS 이름 끝의 점은 완전한 절대 이름(FQDN)을 뜻하며 추가 Zone 접미사가 붙지 않게 한다. `@`는 현재 Zone 기점을 나타낸다.

**25.** SOA Serial은 Zone 버전을 나타내며, Zone 변경 시 증가시키면 secondary 권한 서버가 변경 감지에 활용할 수 있다. 단일 실습 서버에서는 그 자체만으로 데이터 질의 성공을 보장하지 않는다.

**26.** 3600초(1시간) 캐시가 가능하다. 서버 부하를 줄일 수 있지만 주소 변경을 클라이언트가 늦게 인지할 수 있다.

**27.** `named.conf`는 서버 옵션과 관리 Zone을 정의한다. Zone 파일은 그 영역의 SOA·NS·A·PTR 같은 실제 레코드를 보유한다.

**28.** `named-checkconf`는 기본 설정 문법, `named-checkzone`은 특정 Zone 문법/구조, `dig`는 DNS 서버에 실제 질의한 네트워크 동작을 확인한다.

**29.** 기본적으로 UDP/TCP 53. 일반 질의에 UDP가 널리 쓰이며, 크기·잘림(TC)·일부 정책·Zone 전송 등의 경우 TCP도 필요하다.

**30.** 첫 명령은 루프백 주소에 바인딩된 서버로 질의, 두 번째는 서버의 VMware NAT IP로 질의한다. 이번 두 질의는 같은 Rocky `named`로 연결됐다.

**31.** 아니다. 그 명령은 특정 DNS 서버를 직접 지정해 질의한 것이다. 운영체제 전체 DNS 설정을 변경하지 않았다.

**32.** `allow-query`는 DNS 질의 허용 대상, `allow-transfer`는 Zone 전송 허용 대상, `allow-update`는 동적 갱신 허용 대상, `recursion no`는 외부 이름을 대신 찾아주는 재귀 역할 비활성화다. 설정의 **실제 적용 여부는 별도 점검**해야 한다.

**33.** 서버 IP·VMware NAT·named 수신 IP/UDP·TCP 53·firewalld·Zone `allow-query`·클라이언트 조회 대상 서버를 순서대로 점검한다.

**34.** NXDOMAIN은 이름 없음, REFUSED는 정책 등에 의한 거부, SERVFAIL은 서버의 질의 처리 실패, timeout은 네트워크/서비스/방화벽 등에서 응답을 못 받은 상황으로 살핀다.

**35.** 완료: SSH 서비스/22번, ED25519 키 인증, SCP 전송, BIND 정/역 Zone 구성과 Linux·Windows A/PTR 조회. 미완료: Telnet 설치(의도 생략), 실제 RDP 연결, SSH 보안 강화 적용, DNS 영구 방화벽·재부팅 검증, 모든 교재 소절의 누락 점검.

---

## 3. 5분 챌린지

- **1분:** TCP 22의 SSH 공개키 인증 과정 설명(서버 키/사용자 키/passphrase까지).
- **1분:** `web.lab.test`를 `192.168.199.132`로 바꾸는 정방향 DNS 원리.
- **1분:** `199.168.192.in-addr.arpa`와 `132 PTR` 역방향 구조.
- **1분:** DNS 장애에서 서비스/포트/방화벽/ACL/Zone 순으로 원인 좁히기.
- **1분:** 지금까지 검증한 기능 vs 미검증 과제를 구분해 말하기.

### 연습 6개

```bash
systemctl is-active sshd
sudo ss -tlnp | grep ':22'
sudo named-checkconf -z
sudo named-checkzone lab.test /var/named/lab.test.zone
dig @192.168.199.132 web.lab.test A +short
dig @192.168.199.132 -x 192.168.199.132 +short
```

## 4. 개인 복습 점검표

- [ ] 책이나 README 없이 질문 35개 중 최소 28개를 정확히 답한다.
- [ ] SSH 호스트 키/사용자 키를 종이에 그림으로 설명한다.
- [ ] Root→TLD→권한 DNS vs 실습 BIND 권한 Zone의 차이를 설명한다.
- [ ] DNS 레코드 최소 6종을 설명한다.
- [ ] Zone 파일의 Serial·TTL·`@`·마지막 점을 설명한다.
- [ ] 보안 설정에 남아 있는 위험과 앞으로 할 작업을 제시한다.
