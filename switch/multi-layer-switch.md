# Multi-Layer Switch (MLS)

Layer 3 기능을 탑재한 스위치로, **Router 없이 VLAN 간 라우팅** 가능

---

## 개념

- L2 Switch + Router 기능을 하나의 장비에서 처리
- **SVI (Switched Virtual Interface)**: VLAN에 가상 IP를 할당하여 게이트웨이로 사용
- **Routed Interface**: 물리 포트를 `no switchport`로 L3 포트로 전환하여 IP 직접 할당

---

## Routed Interface

물리 포트 자체에 IP를 할당하는 방식 — VLAN에 소속되지 않음

```
SW1(config)# ip routing
!
SW1(config)# interface loopback 0
SW1(config-if)# ip address 13.13.10.1 255.255.255.0
!
SW1(config)# interface fastethernet 0/1
SW1(config-if)# no switchport          ← L2 기능 해제 → L3 포트로 전환
SW1(config-if)# ip address 13.13.1.254 255.255.255.0
!
SW1(config)# interface fastethernet 0/24
SW1(config-if)# no switchport
SW1(config-if)# ip address 13.13.12.1 255.255.255.0
```

### 확인

```
SW1# show ip route
     13.0.0.0/24 is subnetted, 3 subnets
C       13.13.1.0 is directly connected, FastEthernet0/1
C       13.13.10.0 is directly connected, Loopback0
C       13.13.12.0 is directly connected, FastEthernet0/24
```

### 실습 예시: 여러 대의 L3 Switch를 Routed Interface로 연결

> 조건: SW1 - SW2 - SW3 - R4가 각각 Routed Interface로 직접 연결되어 있고, 각 스위치는 서로의 직결 네트워크만 알고 있으면 된다(라우팅 프로토콜 없이 직결 경로만 확인).

```
# SW2 (SW1, SW3, R4와 각각 Routed Interface로 연결)
SW2(config)# ip routing
SW2(config)# interface loopback 0
SW2(config-if)# ip address 13.13.20.2 255.255.255.0
SW2(config)# interface fastethernet 0/10
SW2(config-if)# no switchport
SW2(config-if)# ip address 13.13.22.2 255.255.255.0
SW2(config)# interface fastethernet 0/22
SW2(config-if)# no switchport
SW2(config-if)# ip address 13.13.23.2 255.255.255.0
SW2(config)# interface fastethernet 0/24
SW2(config-if)# no switchport
SW2(config-if)# ip address 13.13.12.2 255.255.255.0
```

각 포트가 `no switchport`로 L3 포트가 되면서 서로 다른 서브넷으로 직결되므로, `show ip route`에는 4개의 Connected 경로가 별도로 나타난다. SW2에서 SW1(13.13.12.1) · R4(13.13.22.4) 방향은 `ping`이 성공하지만, SW1과 SW3처럼 서로 직결되지 않은 네트워크끼리는(예: SW2 → 13.13.13.3) 라우팅 프로토콜 없이는 통신되지 않는다.

---

## SVI (Switched Virtual Interface)

VLAN에 가상 IP를 할당하여 **게이트웨이** 또는 **스위치 관리 IP**로 사용

### SVI 용도

1. **Switch 관리용 IP** — Telnet, SSH, Ping 등 원격 관리 접속
2. **VLAN의 Default Gateway** — MLS에서 VLAN 간 라우팅 수행

### SVI 설정

```
SW1(config)# ip routing
!
SW1(config)# interface vlan 11
SW1(config-if)# ip address 13.13.11.1 255.255.255.0
!
SW1(config)# interface vlan 12
SW1(config-if)# ip address 13.13.12.1 255.255.255.0
```

> ⚠️ `ip routing`을 반드시 먼저 활성화해야 SVI 간 라우팅이 가능하다

### 실습 예시: PC와 Gateway(SVI)가 통신되지 않는 경우

> 조건: SW1에 VLAN 11용 SVI(13.13.11.1)를 만들었는데, 내부 PC에서 Gateway(13.13.11.1)로 ping이 되지 않는다.

```
SW1(config)# interface range fa0/1 - 2
SW1(config-if-range)# switchport mode access
SW1(config-if-range)# switchport access vlan 11
```

- 스위치 포트는 기본적으로 **VLAN 1**에 access되어 있다. PC가 연결된 포트를 SVI와 같은 VLAN(11)으로 지정해주지 않으면, PC는 여전히 VLAN 1에 속해 있어 VLAN 11의 Gateway와 서로 다른 브로드캐스트 도메인에 있는 것과 같다.
- 포트를 `switchport access vlan 11`로 맞춰준 뒤에야 `SW1_내부PC> ping 13.13.11.1`이 정상 응답한다.

### SVI + VTP + OSPF 통합 구성 예시

```
# SW1 — VTP Server, OSPF 참여
SW1(config)# ip routing
!
SW1(config)# vtp mode server
SW1(config)# vtp domain SOL
SW1(config)# vtp password Sol1234
!
SW1(config)# vlan 11
SW1(config)# vlan 12
!
SW1(config)# interface fastethernet 0/24
SW1(config-if)# switchport trunk encapsulation dot1q
SW1(config-if)# switchport mode trunk
!
SW1(config)# interface range fa0/1 - 2
SW1(config-if-range)# switchport mode access
SW1(config-if-range)# switchport access vlan 11
!
SW1(config)# interface vlan 11
SW1(config-if)# ip address 13.13.11.1 255.255.255.0
!
SW1(config)# interface vlan 12
SW1(config-if)# ip address 13.13.12.1 255.255.255.0
!
SW1(config)# interface loopback 0
SW1(config-if)# ip address 13.13.1.1 255.255.255.0
!
SW1(config)# router ospf 1
SW1(config-router)# router-id 1.1.1.1
SW1(config-router)# passive-interface default
SW1(config-router)# no passive-interface vlan 12
SW1(config-router)# network 13.13.1.0 0.0.0.255 area 0
SW1(config-router)# network 13.13.11.0 0.0.0.255 area 0
SW1(config-router)# network 13.13.12.0 0.0.0.255 area 0
```

### OSPF 인접성 확인

```
SW1# show ip ospf neighbor
Neighbor ID     Pri   State       Dead Time   Address       Interface
2.2.2.2           1   FULL/BDR    00:00:33    13.13.12.2    Vlan12

SW1# show ip route
C    13.13.1.0/24 is directly connected, Loopback0
C    13.13.11.0/24 is directly connected, Vlan11
C    13.13.12.0/24 is directly connected, Vlan12
O    13.13.13.0/24 [110/2] via 13.13.12.2, Vlan12
O    13.13.14.0/24 [110/3] via 13.13.12.2, Vlan12
```

---

## 외부 Router 연결 (Uplink)

MLS에서 SVI로 Inter-VLAN 처리, 외부 네트워크는 Routed Port로 Router에 연결

```
MLS(config)# ip routing
!
MLS(config)# interface fastethernet 0/24
MLS(config-if)# no switchport
MLS(config-if)# ip address 10.10.10.2 255.255.255.252
!
MLS(config)# ip route 0.0.0.0 0.0.0.0 10.10.10.1
```

---

## Sub-interface 방식과 비교

| 구분 | Router on a Stick | Multi-Layer Switch |
|------|-------------------|-------------------|
| 장비 | Router + Switch | MLS 단독 |
| 성능 | 소프트웨어 처리 | 하드웨어 처리 (빠름) |
| 비용 | 저렴 | 높음 |
| 링크 | Trunk 1개 | SVI 사용 |
| 확장성 | 제한적 | 우수 |

---

## 정보 확인

```
MLS# show ip routing
MLS# show ip route
MLS# show interface vlan 10
MLS# show ip interface brief
MLS# show vtp status
```

---

## IOS XE 16.x / 17.x 최신 트렌드

### SDM Template — 포워딩 테이블 최적화 (`IOS XE 16.x+`)

L3 Switch의 하드웨어 TCAM을 IPv4/IPv6/ACL/Multicast 중 어디에 얼마나 할당할지 결정.

```
! 현재 SDM Template 확인
SW# show sdm prefer
!  The current template is "default" template.
!  The selected template optimizes the resources in the switch to support
!  this level of features for 8 routers, 1024 VLANs, 8000 unicast routes, ...

! IPv6 라우팅 활성화 시 템플릿 변경 필요
SW(config)# sdm prefer dual-ipv4-and-ipv6 default   ! IPv4+IPv6 동시 지원
SW# reload                                           ! 재부팅 필요

! 라우팅 최적화 (대규모 라우팅 테이블 환경)
SW(config)# sdm prefer routing                       ! 라우팅 TCAM 확대
```

### IPv6 Inter-VLAN 라우팅 (`IOS XE 16.x+`)

```
! IPv6 SVI 설정 (IOS XE 16.x+)
SW(config)# ipv6 unicast-routing

SW(config)# interface vlan 10
SW(config-if)# ipv6 address 2001:db8:10::1/64
SW(config-if)# ipv6 nd prefix 2001:db8:10::/64     ! RA 광고

SW(config)# interface vlan 20
SW(config-if)# ipv6 address 2001:db8:20::1/64
```

### VLAN ACL (VACL) + VLAN Filter (`IOS XE 16.x+`)

SVI ACL은 라우팅 트래픽만 제어. VACL은 **같은 VLAN 내 스위칭 트래픽도 제어** 가능.

```
! VACL 설정 (IOS XE 16.x+)
! 1단계: VLAN Access-map 생성
vlan access-map VLAN10_FILTER 10
 match ip address 101             ! ACL 101에 매칭
 action drop                      ! 차단

vlan access-map VLAN10_FILTER 20
 action forward                   ! 나머지 허용

! 2단계: VLAN에 적용
vlan filter VLAN10_FILTER vlan-list 10

! 확인
SW# show vlan access-map
SW# show vlan filter
```

> IOS 15.x Phase 2 자료에서는 SVI에 `ip access-group`만 사용.  
> IOS XE 16.x VACL은 같은 VLAN 내 PC 간 트래픽도 제어 가능 — 보안 격리에 유효.
