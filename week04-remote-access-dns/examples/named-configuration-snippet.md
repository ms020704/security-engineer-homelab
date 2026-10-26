# named.conf 설정 구조 예시 — 그대로 덮어쓰지 말 것

> 아래는 실습 당시 설명을 바탕으로 재구성한 **샘플 조각**이다. 현재 시스템의 `/etc/named.conf` 원본 내용 전체가 아니다. `options {}`를 중복 작성하면 오류가 날 수 있으므로 **기존 options 블록 내부에 필요한 항목만 수정**하고 Zone 선언은 블록 외부에 추가해야 한다.

## 1. 기존 `options { ... };`에 적용할 설정 예시

```conf
// 서버의 다른 기본 옵션은 보존한다.
listen-on port 53 { 127.0.0.1; 192.168.199.132; };
recursion no;
```

- `listen-on`: 127.0.0.1과 NAT 네트워크 주소에서 TCP/UDP 53 질의를 수신.
- `recursion no;`: 공개 재귀 리졸버로 동작하지 않도록 제한하는 구성 예시.
- 실제로 적용됐는지 `grep`/`named-checkconf`로 별도 점검한다.

## 2. `options` 블록 바깥의 Zone 선언 예시

```conf
zone "lab.test" IN {
    type master;
    file "lab.test.zone";
    allow-query { localhost; 192.168.199.0/24; };
    allow-update { none; };
    allow-transfer { none; };
};

zone "199.168.192.in-addr.arpa" IN {
    type master;
    file "199.168.192.rev";
    allow-query { localhost; 192.168.199.0/24; };
    allow-update { none; };
    allow-transfer { none; };
};
```

### 검증 명령

```bash
sudo cp -a /etc/named.conf /etc/named.conf.bak-before-change
sudo named-checkconf -z
sudo named-checkzone lab.test /var/named/lab.test.zone
sudo named-checkzone 199.168.192.in-addr.arpa /var/named/199.168.192.rev
sudo systemctl reload named
```

실습망 IP가 달라지면 레코드와 ACL·리스닝 주소를 조정해야 한다. 서버가 인터넷과 연결된 환경에서는 BIND 및 firewalld의 클라이언트 접근 허용 범위를 더 제한할 것.
