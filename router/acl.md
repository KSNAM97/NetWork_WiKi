# ACL (Access Control List)

네트워크 트래픽을 필터링하여 허용 또는 차단하는 기능

---

## Standard ACL

- **출발지 IP 주소**만을 기준으로 필터링
- ACL 번호: **1 ~ 99**, 1300 ~ 1999

```
Router(config)# access-list [1-99] permit/deny [Source-IP] [Wildcard]
Router(config-if)# ip access-group [ACL번호] in/out
```

### 예시

```
R1(config)# access-list 10 deny 192.168.10.0 0.0.0.255
R1(config)# access-list 10 permit any
!
R1(config)# interface fastethernet 0/0
R1(config-if)# ip access-group 10 in
```

> ⚠️ Standard ACL은 목적지에 **최대한 가깝게** 적용 (출발지 정보만 있어 범위 최소화 필요)

---

## Extended ACL

- **출발지 IP, 목적지 IP, Protocol, Port** 번호를 기준으로 필터링
- ACL 번호: **100 ~ 199**, 2000 ~ 2699

```
Router(config)# access-list [100-199] permit/deny [Protocol] [Src-IP] [Wildcard] [Dst-IP] [Wildcard] [eq Port]
Router(config-if)# ip access-group [ACL번호] in/out
```

### 예시

```
R1(config)# access-list 101 deny tcp 192.168.10.0 0.0.0.255 host 10.10.10.1 eq 80
R1(config)# access-list 101 deny tcp 192.168.10.0 0.0.0.255 host 10.10.10.1 eq 443
R1(config)# access-list 101 permit ip any any
!
R1(config)# interface fastethernet 0/0
R1(config-if)# ip access-group 101 in
```

> ✅ Extended ACL은 출발지에 **최대한 가깝게** 적용 (불필요한 트래픽을 초기에 차단)

**예시 문제**: R2 내부 네트워크 13.13.12.0/24로 들어오는 트래픽 중, 150.1.13.4에서 오는 Telnet과 150.3.13.5에서 오는 HTTP는 차단하고, 출발지 13.13.11.0/24 → 목적지 13.13.12.0/24 구간의 ICMP도 차단해야 한다. 나머지 트래픽은 모두 허용. 이 ACL을 R2의 어느 인터페이스에, `in`과 `out` 중 어느 방향으로 적용해야 할까?

- 트래픽이 **외부에서 R2로 들어오는** 시점에 걸러야 하므로, 외부와 연결된 Serial 인터페이스에 **`in`** 방향으로 적용한다. (반대로 `out`으로 걸면 R2 내부에서 나가는 트래픽 기준으로 검사되어 조건과 어긋난다.)

```
R2(config)# access-list 101 deny tcp host 150.1.13.4 13.13.12.0 0.0.0.255 eq 23
R2(config)# access-list 101 deny tcp host 150.3.13.5 13.13.12.0 0.0.0.255 eq 80
R2(config)# access-list 101 deny icmp 13.13.11.0 0.0.0.255 13.13.12.0 0.0.0.255
R2(config)# access-list 101 permit ip any any
!
R2(config)# interface serial 1/0.123     ! 외부(WAN)와 연결된 인터페이스
R2(config-if)# ip access-group 101 in    ! 인터페이스로 "들어오는" 트래픽 기준 — in
```

확인: `R4# telnet 13.13.12.2` → 접속 실패(`Destination unreachable`), `R1# ping 13.13.12.2 source 13.13.11.1` → 실패. 만약 `ip access-group 101 out`으로 잘못 적용했다면 이 ACL은 R2에서 나가는 트래픽에는 영향을 주지 못해 조건을 만족하지 못한다.

---

## Named ACL

ACL 번호 대신 이름을 사용하는 방식 — 가독성 및 수정 용이

```
R1(config)# ip access-list standard BLOCK_HOST
R1(config-std-nacl)# deny 192.168.10.1
R1(config-std-nacl)# permit any
!
R1(config)# ip access-list extended ALLOW_WEB
R1(config-ext-nacl)# permit tcp any any eq 80
R1(config-ext-nacl)# permit tcp any any eq 443
R1(config-ext-nacl)# deny ip any any
```

---

## 실습 예시

### Extended ACL — 복합 조건 차단

```
# R4의 Telnet, R5의 HTTP, 특정 네트워크 ICMP 차단
R2(config)# access-list 101 deny tcp host 150.1.13.4 13.13.12.0 0.0.0.255 eq 23
R2(config)# access-list 101 deny tcp host 150.3.13.5 13.13.12.0 0.0.0.255 eq 80
R2(config)# access-list 101 deny icmp 13.13.11.0 0.0.0.255 13.13.12.0 0.0.0.255
R2(config)# access-list 101 permit ip any any
!
R2(config)# interface serial 1/0.123
R2(config-if)# ip access-group 101 in
```

### log-input 옵션 — 차단 시 수신 인터페이스와 로그 출력

```
R2(config)# access-list 101 deny tcp host 150.1.13.4 13.13.12.0 0.0.0.255 eq 23 log-input
R2(config)# access-list 101 deny icmp 13.13.11.0 0.0.0.255 13.13.12.0 0.0.0.255 log-input
R2(config)# access-list 101 permit ip any any
```

로그 출력 예시:
```
%SEC-6-IPACCESSLOGP: list 101 denied tcp 150.1.13.4(16795) (Serial1/0.123) -> 13.13.12.2(23), 1 packet
```

### ACL 삭제

```
# Numbered ACL
R(config)# no access-list 101
R(config-if)# no ip access-group 101 in

# Named ACL
R(config)# no ip access-list extended ACL
R(config-if)# no ip access-group ACL in
```

### ICMP echo-reply 허용 패턴

외부에서의 ICMP 요청은 차단하되, 내부에서 시작한 ICMP 통신의 응답은 허용하는 패턴

```
R4(config)# access-list 102 permit icmp any 13.13.4.0 0.0.0.255 echo-reply
R4(config)# access-list 102 permit icmp any 13.13.14.0 0.0.0.255 echo-reply
R4(config)# access-list 102 deny icmp any 13.13.4.0 0.0.0.255
R4(config)# access-list 102 deny icmp any 13.13.14.0 0.0.0.255
R4(config)# access-list 102 permit ip any any
!
R4(config)# interface fastethernet 0/0
R4(config-if)# ip access-group 102 in
```

> `echo-reply`: ICMP Type 0 (응답 패킷만 허용). 외부에서 새로 시작하는 ICMP echo(Type 8)는 차단됨.

### IP Fragment 차단

MTU보다 큰 패킷이 분할(Fragment)되어 전송될 때 분할된 패킷을 차단하는 기능

```
R4(config)# access-list 103 deny icmp any host 13.13.4.4 fragment
R4(config)# access-list 103 permit ip any any
!
R4(config)# interface fastethernet 0/0
R4(config-if)# ip access-group 103 in
```

> `fragment` 키워드: 첫 번째 Fragment가 아닌 후속 Fragment 패킷만 차단  
> IP Fragment DoS 공격 방어에 사용됨

### TCP established / ack — 내부 시작 통신만 허용

```
# 방법 1: ack 키워드 (ACK 비트가 설정된 패킷만 허용 — SYN만 차단)
R2(config)# access-list 104 permit tcp any 13.13.12.0 0.0.0.255 ack
R2(config)# access-list 104 deny tcp any 13.13.12.0 0.0.0.255
R2(config)# access-list 104 permit ip any any

# 방법 2: established 키워드 (ACK 또는 RST 비트 설정된 패킷만 허용)
R2(config)# access-list 104 permit tcp any 13.13.12.0 0.0.0.255 established
R2(config)# access-list 104 deny tcp any 13.13.12.0 0.0.0.255
R2(config)# access-list 104 permit ip any any
```

> `established`: TCP 세션 연결 후 응답 트래픽만 허용 (SYN 패킷 차단)

**예시 문제**: R2에서 외부 네트워크가 내부 네트워크 13.13.2.0/24, 13.13.12.0/24로 접근하는 TCP 트래픽은 차단해야 하지만, 13.13.12.0/24가 **먼저 외부로** 나가는 TCP 통신(및 그 응답)은 허용되어야 한다. 단순히 `deny tcp any 13.13.12.0 0.0.0.255`만 넣으면 13.13.12.0/24가 외부로 보낸 요청의 **응답 트래픽**(SA/DA가 뒤바뀐 SYN,ACK)까지 막혀버려 내부 → 외부 통신 자체가 끊긴다. 어떻게 설정해야 하는가?

```
! ack(또는 established)로 응답 트래픽만 먼저 허용한 뒤 신규 요청을 차단
R2(config)# access-list 104 permit tcp any 13.13.2.0 0.0.0.255 established
R2(config)# access-list 104 permit tcp any 13.13.12.0 0.0.0.255 established
R2(config)# access-list 104 deny tcp any 13.13.2.0 0.0.0.255
R2(config)# access-list 104 deny tcp any 13.13.12.0 0.0.0.255
R2(config)# access-list 104 permit ip any any
!
R2(config)# interface serial 1/0.123
R2(config-if)# ip access-group 104 in
```

확인: `R4# telnet 13.13.2.2` → 외부에서 시작한 세션은 실패, `R2# telnet 13.13.4.4` → R2 내부에서 시작한 세션(및 그 응답)은 정상 연결. `established`/`ack` 매칭 라인을 반드시 `deny`보다 **위**에 둬야 응답 트래픽이 먼저 허용된다.

---

## Reflective ACL

내부에서 시작된 세션의 응답 트래픽만 허용하고 외부 시작 트래픽은 차단
- `established` 키워드와 달리 **TCP/UDP/ICMP 모두** 처리 가능
- 트래픽 출력 시 State Table이 자동 생성되어 응답 트래픽 허용

### 동작 방식

1. 내부 → 외부 트래픽이 `reflect` 적용된 ACL에 일치하면 임시 ACL 항목 자동 생성
2. 외부 → 내부 트래픽이 `evaluate`로 임시 ACL과 비교
3. 내부에서 시작된 통신의 응답이면 허용, 외부에서 새로 시작된 통신은 차단

### 설정 예시

```
! 내부 → 외부 방향 (out): reflect로 상태 테이블 생성
ip access-list extended IN-OUT
 permit tcp any any reflect REFLECTIVE_ACL
 permit udp any any reflect REFLECTIVE_ACL
 permit icmp any any reflect REFLECTIVE_ACL

! 외부 → 내부 방향 (in): evaluate로 상태 테이블 검사
ip access-list extended OUT-IN
 permit udp any eq 520 any eq 520    ! RIP 허용 (특수 허용 규칙)
 evaluate REFLECTIVE_ACL             ! 내부에서 시작된 통신의 응답만 허용

! 외부 연결 인터페이스에 적용
interface serial 1/0.123
 ip access-group IN-OUT out
 ip access-group OUT-IN in
```

### 확인

```
R# show access-list
! Reflective ACL 임시 항목 확인:
! Reflexive IP access list REFLECTIVE_ACL
!      permit icmp host 13.13.5.5 host 13.13.4.4  (20 matches) (time left 298)
```

---

## Dynamic ACL (Lock & Key)

외부 사용자가 Telnet으로 **인증**한 후에만 내부 네트워크 접근을 허용하는 기능
- 인증 후 해당 사용자 IP 주소에 대한 임시 ACL 항목이 자동 추가됨
- 설정한 timeout 후 임시 항목 자동 삭제

### 설정 예시

```
! 인증 계정 설정
username soladmin privilege 15 password solcisco
username soladmin autocommand access-enable host   ! 인증 후 임시 ACL 자동 생성

line vty 0 4
 login local

! ACL 설정
access-list 105 permit udp any eq 520 any eq 520    ! RIP 허용
access-list 105 permit tcp any host 13.13.3.3 eq 23 ! 인증용 Telnet 허용
access-list 105 dynamic SOL_DYNAMIC timeout 5 permit ip any 13.13.15.0 0.0.0.255
!   ↑ timeout 5 = 인증 후 5분간 유지되는 임시 허용 ACL

! 외부 인터페이스에 적용
interface serial 1/0.123
 ip access-group 105 in
```

### 동작 흐름

```
1. 외부 사용자가 R3의 Telnet(13.13.3.3:23)으로 접속하여 인증
   → autocommand access-enable host 실행 → Telnet 세션 자동 종료
2. 사용자 IP 주소를 허용하는 임시 ACL 항목 자동 생성
3. 이후 해당 IP에서 13.13.15.0/24로의 트래픽 허용
4. timeout 5분 경과 후 임시 항목 자동 삭제
```

---

## ACL 규칙

1. **순서대로 처리** — 위에서 아래로, 첫 번째 일치 항목에서 처리 종료
2. **암묵적 Deny** — ACL 마지막에 `deny any`가 자동 추가
3. **새 항목은 끝에 추가** — 번호 ACL은 중간 삽입 불가 (Named ACL은 가능)

---

## Time-range ACL

ACL이 **특정 날짜/시간에만 동작**하도록 제한하는 기능

### Periodic (반복 시간대)

```
R(config)# time-range WORK_TIME
R(config-time-range)# periodic weekdays 08:00 to 18:00
R(config-time-range)# periodic weekend 10:00 to 16:00
!
R(config)# access-list 101 permit tcp any host 10.10.10.1 eq 23 time-range WORK_TIME
R(config)# access-list 101 permit tcp any host 10.10.10.1 eq 80 time-range WORK_TIME
```

요일 키워드: `monday`, `tuesday`, `wednesday`, `thursday`, `friday`, `saturday`, `sunday`, `weekdays`, `weekend`, `daily`

### Absolute (특정 기간)

```
R(config)# time-range PROJECT_PERIOD
R(config-time-range)# absolute start 09:30 01 July 2026 end 18:30 29 August 2026
!
R(config)# access-list 103 permit tcp any host 10.10.10.1 eq 23 time-range PROJECT_PERIOD
```

### 확인

```
R# show access-lists
    20 permit tcp any host 13.13.4.4 eq telnet time-range WORK_TIME (active)     ← 시간 범위 안
    20 permit tcp any host 13.13.4.4 eq telnet time-range WORK_TIME (inactive)   ← 시간 범위 밖

R# show time-range
```

---

## IOS XE 16.x / 17.x 최신 트렌드 — Object-Group ACL

> `IOS XE 16.x+` — 출처: [Cisco IOS XE 17.x Object Groups for ACLs](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/sec-vpn/b-security-vpn/m_sec_zbf_ogacl-xe.html)  
> 개별 ACE 대신 그룹으로 관리 → 설정 간소화, NVRAM 절약, 가독성 향상

### Network Object Group (IP 주소 그룹화)

```
! 서버 그룹 정의
object-group network SERVERS
 host 10.10.10.1                     ! 단일 호스트
 host 10.10.10.2
 192.168.20.0 255.255.255.0          ! 서브넷 범위

! 클라이언트 그룹 정의
object-group network CLIENTS
 192.168.10.0 255.255.255.0
```

### Service Object Group (포트 그룹화)

```
! 웹 서비스 포트 그룹 정의
object-group service WEB_PORTS
 tcp destination eq 80               ! HTTP
 tcp destination eq 443              ! HTTPS
```

### Object-Group 기반 ACL 적용

```
ip access-list extended ALLOW_WEB
 ! CLIENTS 그룹에서 SERVERS 그룹으로의 WEB_PORTS 트래픽 허용
 permit tcp object-group CLIENTS object-group SERVERS object-group WEB_PORTS
 deny ip any any log

interface GigabitEthernet 0/1
 ip access-group ALLOW_WEB in
```

> 기존 방식 대비: CLIENTS(1개 네트워크) × SERVERS(3개 호스트) × WEB_PORTS(2개 포트) = 6줄 → **1줄**로 축약

---

## TCP Intercept

SYN Flooding(DoS 공격)으로 서버의 TCP 세션이 Half-open 상태로 가득 차는 현상을 방어하는 기능

### 동작 모드 비교

| 모드 | 동작 방식 | 특징 |
|------|-----------|------|
| **intercept** | 라우터가 3-way Handshake를 클라이언트 대신 처리 후 서버에 연결 | ACK 미수신 시 30초 후 RST 전송 |
| **watch** | TCP 세션을 감시하여 30초 내 연결 실패 시 서버에 RST 전송 | 라우터가 직접 관여하지 않음 |
| **drop** | 1분 내 1100개 불완전 연결 발생 시 순차 삭제 | 공격 중 대응 모드 |

### Intercept Mode 설정 (라우터가 직접 3-way 처리)

```
! 보호 대상 서버를 ACL로 정의
ip access-list extended TCP_INTERCEPT
 permit tcp any host 211.241.228.2   ! 해당 서버로 향하는 TCP 세션 감시

! TCP Intercept 활성화
ip tcp intercept list TCP_INTERCEPT
ip tcp intercept mode intercept
ip tcp intercept connection-timeout 30    ! ACK 대기 시간 (기본 30초)
```

### Intercept Mode 동작 흐름

```
[정상 연결]
Client → SYN → Router (대리 처리)
Client ← SYN+ACK ← Router
Client → ACK → Router → Router가 서버와 실제 TCP 연결
→ Client와 Server 세션 연결

[비정상 연결 — ACK 미수신]
Client → SYN → Router
Client ← SYN+ACK ← Router
(ACK 없음) → 30초 후 RST 전송 → 세션 강제 종료
```

### Watch Mode 설정 (감시 후 타임아웃 처리)

```
ip access-list extended TCP_INTERCEPT
 permit tcp any host 211.241.228.2

ip tcp intercept list TCP_INTERCEPT
ip tcp intercept mode watch
ip tcp intercept watch-timeout 30     ! 감시 타임아웃 (기본 30초)
```

### Debug 출력 예시

```
R1# debug ip tcp intercept

! 정상 연결 (Intercept Mode)
INTERCEPT: new connection (13.13.4.4:31458 SYN -> 211.241.228.2:23)
INTERCEPT(*): (13.13.4.4:31458 <- ACK+SYN 211.241.228.2:23)    ← 라우터 대리 응답
INTERCEPT: 1st half of connection established
INTERCEPT(*): (13.13.4.4:31458 SYN -> 211.241.228.2:23)         ← 서버에 재연결
INTERCEPT: 2nd half of connection established                    ← 완전 연결

! 비정상 연결 (ACK 미수신 → RST)
INTERCEPT: new connection (13.13.4.4:27460 SYN -> 211.241.228.2:23)
INTERCEPT(*): SYNRCVD retransmit 1~4                             ← 재전송
INTERCEPT: SYNRCVD retransmitting too long
INTERCEPT(*): (13.13.4.4:27460 <- RST 211.241.228.2:23)         ← 강제 종료
```

---

## 정보 확인 (검증 명령어)

```
! 모든 ACL 항목 + 각 항목별 매칭된 패킷 수 — 0이면 트래픽이 통과하지 않은 것
R# show access-lists

! IP ACL만 필터링 — show access-lists와 동일하나 non-IP ACL 제외
R# show ip access-lists

! 인터페이스에 어떤 ACL이 적용됐는지 (in/out 방향 포함)
R# show interface fa0/0

! 인터페이스별 ACL 적용 현황 요약 — 모든 인터페이스를 한번에 확인
R# show ip interface brief | include ACL
R# show ip interface fa0/0     ← Inbound/Outbound access list 항목 확인

! Time-range 현재 상태 확인 — Active/Inactive 여부
R# show time-range

! Object-group 내용 확인 (IOS XE)
R# show object-group
```

| 명령어 | 주로 확인하는 것 |
|--------|----------------|
| `show access-lists` | 각 ACE별 hit 횟수 — 차단/허용 확인 |
| `show ip interface fa0/0` | 해당 포트에 ACL이 in/out 방향으로 적용됐는지 |
| `show time-range` | 시간 기반 ACL의 현재 active/inactive 상태 |
| `show object-group` | Object-Group에 포함된 IP/포트 목록 확인 |
