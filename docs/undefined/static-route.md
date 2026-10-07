# Static Route (정적 라우팅)

관리자가 목적지 네트워크까지의 경로를 직접 입력하는 방식

---

## 특징

| 항목 | 내용 |
|------|------|
| 용도 | 소규모 네트워크, 변경이 없는 환경 |
| 장점 | 패킷 이동 경로 정확히 제어 가능, CPU 효율적 |
| 단점 | 경로 변화 시 수동 업데이트 필요, 이중화 자동 복구 불가 |
| AD | **1** (Connected=0, Dynamic보다 우선) |

---

## 설정 방법

```
! 방법 1: Next-hop IP 지정
Router(config)# ip route [목적지 Network] [SubnetMask] [Next-hop IP]

! 방법 2: 출구 인터페이스 지정
Router(config)# ip route [목적지 Network] [SubnetMask] [Connected Interface]

! Default Route (모든 목적지)
Router(config)# ip route 0.0.0.0 0.0.0.0 [Next-hop IP]
```

---

## 실습 예시

```
                192.168.12.0/24          192.168.23.0/24
         R1---------------------R2---------------------R3
          |                      |                      |
   192.168.1.0/24         192.168.2.0/24         192.168.3.0/24
```

### 사전 설정 (Pre-config) — 인터페이스 IP 할당

Static Route를 설정하기 전, 먼저 각 라우터의 인터페이스에 IP를 할당하고 WAN 구간을 활성화해야 한다. LAN 구간은 PC가 첫 번째 주소, Gateway가 마지막 주소를 사용하며, WAN 구간은 HDLC 캡슐화에 대역폭 64K를 사용한다.

```
# R1
R1(config)# interface fastethernet 0/0
R1(config-if)# no shutdown
R1(config-if)# ip address 192.168.1.254 255.255.255.0
!
R1(config)# interface serial 1/0
R1(config-if)# no shutdown
R1(config-if)# encapsulation hdlc
R1(config-if)# bandwidth 64
R1(config-if)# ip address 192.168.12.1 255.255.255.0

# R2
R2(config)# interface fastethernet 0/0
R2(config-if)# no shutdown
R2(config-if)# ip address 192.168.2.254 255.255.255.0
!
R2(config)# interface serial 1/1
R2(config-if)# no shutdown
R2(config-if)# encapsulation hdlc
R2(config-if)# bandwidth 64
R2(config-if)# clock rate 64000          ! DCE 측 — 클럭 제공
R2(config-if)# ip address 192.168.12.2 255.255.255.0
!
R2(config)# interface serial 1/0
R2(config-if)# no shutdown
R2(config-if)# encapsulation hdlc
R2(config-if)# bandwidth 64
R2(config-if)# clock rate 64000
R2(config-if)# ip address 192.168.23.2 255.255.255.0

# R3
R3(config)# interface fastethernet 0/0
R3(config-if)# no shutdown
R3(config-if)# ip address 192.168.3.254 255.255.255.0
!
R3(config)# interface serial 1/1
R3(config-if)# no shutdown
R3(config-if)# encapsulation hdlc
R3(config-if)# bandwidth 64
R3(config-if)# ip address 192.168.23.3 255.255.255.0
```

> `clock rate`는 Serial 케이블의 **DCE(Data Communication Equipment)** 측 인터페이스에서만 설정한다. Packet Tracer 등 시뮬레이터에서 DCE 쪽을 확인 후 설정해야 링크가 정상적으로 Up 된다.

### Next-hop IP 방식

```
# R1
ip route 192.168.2.0 255.255.255.0 192.168.12.2   ! R2 LAN → R2를 통해
ip route 192.168.3.0 255.255.255.0 192.168.12.2   ! R3 LAN → R2를 통해
ip route 192.168.23.0 255.255.255.0 192.168.12.2  ! WAN 구간 → R2를 통해

# R2
ip route 192.168.1.0 255.255.255.0 192.168.12.1   ! R1 LAN
ip route 192.168.3.0 255.255.255.0 192.168.23.3   ! R3 LAN

# R3
ip route 192.168.1.0 255.255.255.0 192.168.23.2   ! R1 LAN → R2를 통해
ip route 192.168.2.0 255.255.255.0 192.168.23.2   ! R2 LAN
ip route 192.168.12.0 255.255.255.0 192.168.23.2  ! WAN 구간
```

### 출구 인터페이스 방식

```
# R1 (Serial 포트로 지정)
ip route 192.168.2.0 255.255.255.0 serial 1/0
ip route 192.168.3.0 255.255.255.0 serial 1/0
ip route 192.168.23.0 255.255.255.0 serial 1/0
```

> ⚠️ Ethernet 인터페이스에는 Interface 방식 사용 주의 — Proxy ARP 문제 발생 가능. Serial(P2P)에서는 양쪽 무관.

---

## 라우팅 테이블 확인

```
R1# show ip route
C    192.168.1.0/24 is directly connected, FastEthernet0/0  ← C = Connected
S    192.168.2.0/24 [1/0] via 192.168.12.2                 ← S = Static, [AD/Metric]
S    192.168.3.0/24 [1/0] via 192.168.12.2
C    192.168.12.0/24 is directly connected, Serial1/0
S    192.168.23.0/24 [1/0] via 192.168.12.2
```

### 인터페이스 방식 라우팅 테이블

```
R1# show ip route
S    192.168.2.0/24 is directly connected, Serial1/0   ← 인터페이스 방식은 "directly connected"로 표시
```

---

## Default Route

모든 목적지에 대한 경로 — Dynamic Protocol이 없는 Edge 라우터에서 Internet 방향으로 사용

```
! Default Route — 0.0.0.0/0 = 모든 목적지에 매칭
ip route 0.0.0.0 0.0.0.0 192.168.12.1    ! Next-hop IP 지정
ip route 0.0.0.0 0.0.0.0 serial 1/0      ! 또는 인터페이스 지정

! OSPF/EIGRP로 Default Route 재배포
router ospf 1
 default-information originate            ! 자신의 Default Route를 OSPF로 광고
```

**예시 문제**: R1에서 인터넷으로 나가는 주 경로는 Serial 1/0(192.168.12.2, ISP-A)이고, 이 회선이 끊기면 보조 회선인 Serial 1/1(192.168.13.3, ISP-B)로 자동 전환되어야 한다. 두 개의 Default Route를 동시에 넣으면 EIGRP/OSPF처럼 부하분산이 아니라 둘 다 AD가 같아 예측 불가능한 동작이 될 수 있는데, "평소엔 A만 쓰고 A가 죽으면 B" 조건을 만들려면?

```
! 주 경로 — 기본 AD 1 그대로 사용
R1(config)# ip route 0.0.0.0 0.0.0.0 192.168.12.2

! 보조 경로 — AD를 1보다 높은 값(예: 5)으로 지정해 "Floating" Static Route로 등록
R1(config)# ip route 0.0.0.0 0.0.0.0 192.168.13.3 5
```

확인: 평소 `R1# show ip route`에는 AD가 더 낮은(1) 주 경로만 올라오고, 보조 경로는 라우팅 테이블에 보이지 않는다(백업 대기 상태). Serial 1/0을 `shutdown`하면 주 경로가 사라지면서 AD 5인 보조 경로가 자동으로 라우팅 테이블에 등록되어 트래픽이 ISP-B로 전환된다.

---

## 정보 확인

```
! 라우팅 테이블 전체 확인 — C/S/R/O/D 코드로 경로 종류 구분
R# show ip route

! Static Route만 필터링
R# show ip route static
```

---

## IOS XE 16.x / 17.x 최신 트렌드

### IP SLA 기반 Floating Static Route (`IOS XE 16.x+`)

단순 AD 값 조정(Floating)이 아닌 **실제 원격 연결성**을 체크하여 경로 전환

```
! 1단계: IP SLA로 Primary 경로 종단 연결성 체크
ip sla 1
 icmp-echo 8.8.8.8 source-interface GigabitEthernet0/1
 frequency 10
ip sla schedule 1 start-time now life forever

! 2단계: SLA 결과를 Track에 연결
track 10 ip sla 1 reachability

! 3단계: Track 연동 Static Route (Primary)
ip route 0.0.0.0 0.0.0.0 203.0.113.1 track 10   ! SLA 실패 시 자동 제거

! 4단계: Floating Static Route (Backup — AD 5로 낮은 우선순위)
ip route 0.0.0.0 0.0.0.0 198.51.100.1 5          ! AD=5 (기본 AD=1보다 높음)
!  → Primary SLA 실패 → Primary 경로 제거 → Backup 자동 활성화
```

> IOS 15.x Phase 2 자료에서는 AD 값만으로 Floating 구성.  
> IOS XE 16.x는 IP SLA로 실제 인터넷 연결 여부까지 체크하여 더 정교한 이중화 가능.

### IPv6 Static Route (`IOS XE 16.x+`)

```
! IPv6 기본 Static Route
ipv6 unicast-routing
ipv6 route 2001:db8:10::/48 2001:db8:12::2

! IPv6 Default Route
ipv6 route ::/0 GigabitEthernet0/1

! IPv6 Floating (AD 지정)
ipv6 route ::/0 2001:db8:backup::1 10
```
