# EtherChannel

여러 물리적 링크를 **하나의 논리적 링크**로 묶어 대역폭 확장 및 이중화를 제공

---

## 개념

- 최대 **8개** 물리 포트를 하나의 논리 채널로 구성
- STP는 EtherChannel을 **단일 링크**로 인식 → 블로킹 없이 모든 링크 사용
- 링크 장애 시 자동으로 다른 링크로 트래픽 분산 → **이중화**

> 💡 STP 환경에서 스위치 간 링크를 추가해도 STP가 나머지를 Blocking 처리하여 실제 사용 링크는 1개로 제한됨 — EtherChannel로 해결

---

## 협상 프로토콜

### PAgP (Port Aggregation Protocol)

- **CISCO 전용**

| 모드 | 동작 |
|------|------|
| **desirable** | PAgP 메시지 능동적 송수신 |
| **auto** | PAgP 메시지 수신 시만 응답 |

> desirable-desirable, desirable-auto 조합 가능 / auto-auto 불가

### LACP (Link Aggregation Control Protocol)

- **IEEE 802.3ad** 표준 — 멀티 벤더 환경에서 사용 가능

| 모드 | 동작 |
|------|------|
| **active** | LACP 메시지 능동적 송수신 |
| **passive** | LACP 메시지 수신 시만 응답 |

> active-active, active-passive 조합 가능 / passive-passive 불가

### On Mode

- 협상 없이 강제로 구성 — 상대방도 반드시 `on` 모드여야 함

---

## L2 EtherChannel 설정

### 설정 방법

```
# PAgP
interface port-channel 12
!
interface range fastethernet 0/23 - 24
 channel-protocol pagp
 channel-group 12 mode desirable

# LACP
interface port-channel 13
!
interface range fastethernet 0/21 - 22
 channel-protocol lacp
 channel-group 13 mode active

# On Mode
interface port-channel 1
!
interface range fastethernet 0/1 - 2
 channel-group 1 mode on
```

### Port-channel에 Trunk 적용

```
interface port-channel 12
 switchport trunk encapsulation dot1q
 switchport mode trunk
!
interface port-channel 13
 switchport trunk encapsulation dot1q
 switchport mode trunk
```

### 실습 구성 예시 (SW1-SW2-SW3)

```
# SW1
interface port-channel 12
interface port-channel 13
!
interface range fa0/23 - 24
 channel-protocol pagp
 channel-group 12 mode desirable
!
interface range fa0/21 - 22
 channel-protocol lacp
 channel-group 13 mode active

# SW2
interface port-channel 12
interface port-channel 23
!
interface range fa0/23 - 24
 channel-protocol pagp
 channel-group 12 mode desirable
!
interface range fa0/19 - 20
 channel-protocol pagp
 channel-group 23 mode desirable

# SW3
interface port-channel 13
interface port-channel 23
!
interface range fa0/21 - 22
 channel-protocol lacp
 channel-group 13 mode active
!
interface range fa0/19 - 20
 channel-protocol pagp
 channel-group 23 mode desirable
```

### 확인

```
SW1# show etherchannel summary
Group  Port-channel  Protocol   Ports
------+-------------+-----------+------
12     Po12(SU)      PAgP       Fa0/23(P) Fa0/24(P)
13     Po13(SU)      LACP       Fa0/21(P) Fa0/22(P)
```

**상태 코드**

| 코드 | 의미 |
|------|------|
| S | Layer 2 |
| U | In use |
| R | Layer 3 |
| P | Bundled in port-channel |

---

## L3 EtherChannel 설정

포트에 `no switchport`를 적용하고 Port-channel에 IP를 부여하는 방식

```
SW1(config)# ip routing
!
SW1(config)# interface range fa0/23 - 24
SW1(config-if-range)# no switchport
!
SW1(config)# interface port-channel 12
SW1(config-if)# no switchport
SW1(config-if)# ip address 192.168.12.1 255.255.255.0
!
SW1(config)# interface range fa0/23 - 24
SW1(config-if-range)# channel-protocol pagp
SW1(config-if-range)# channel-group 12 mode desirable
```

### L3 EtherChannel 확인

```
SW# show etherchannel summary
Group  Port-channel  Protocol   Ports
------+-------------+-----------+------
12     Po12(RU)      PAgP       Fa0/23(P) Fa0/24(P)
```

> `RU` = Routed(R) + In use(U)

---

## 로드 밸런싱

```
SW(config)# port-channel load-balance [방식]
```

| 방식 | 기준 |
|------|------|
| src-mac | 출발지 MAC |
| dst-mac | 목적지 MAC |
| src-dst-mac | 출발지+목적지 MAC |
| src-ip | 출발지 IP |
| dst-ip | 목적지 IP |
| src-dst-ip | 출발지+목적지 IP |

---

## EtherChannel 구성 요구 사항

같은 EtherChannel에 묶이는 모든 포트는 동일해야 함:
- 속도 (Speed) / 듀플렉스 (Duplex)
- VLAN / Trunk 설정
- STP 설정

> ⚠️ **SPAN, Port-Security가 설정된 포트에서는 L2 EtherChannel 사용 불가**

---

## 정보 확인

```
SW# show etherchannel summary
SW# show etherchannel port-channel
SW# show interface port-channel 1
SW# show lacp neighbor
SW# show pagp neighbor
```

---

## IOS XE 16.x / 17.x 최신 트렌드

### LACP 고급 옵션 (`IOS XE 16.x+`)

Phase 2 자료의 기본 LACP 설정에 추가되는 기능들.

```
! LACP Fast Timer — 링크 장애 감지 속도 향상 (기본 30초 → 1초)
interface GigabitEthernet1/0/1
 channel-group 1 mode active
 lacp rate fast                  ! LACP PDU를 1초마다 전송 (기본 30초)

! LACP System Priority — 어떤 스위치가 협상 주도권을 가질지 결정
SW(config)# lacp system-priority 100      ! 낮을수록 우선 (기본 32768)

! LACP Port Priority — 같은 번들 내 우선 포트 지정
interface GigabitEthernet1/0/1
 lacp port-priority 100          ! 낮을수록 우선 활성 포트
```

**예시 문제**: SW1-SW2 구간은 Fa0/23-24를 PAgP(desirable)로, SW1-SW3 구간(SW1의 Fa0/21-22)은 LACP(active)로 각각 EtherChannel을 구성해야 한다. 상대 스위치인 SW2/SW3 쪽 모드를 잘못 맞추면(예: 양쪽 다 auto, 혹은 양쪽 다 passive) 채널이 아예 형성되지 않는데, 어떤 조합이 유효한가?

- PAgP: desirable-desirable, desirable-auto는 성립 / **auto-auto는 성립하지 않음** (양쪽 다 수동적으로 대기만 하므로)
- LACP: active-active, active-passive는 성립 / **passive-passive는 성립하지 않음**

```
! SW1 — SW2 방향 (PAgP desirable), SW3 방향 (LACP active)
interface range fastethernet 0/23 - 24
 channel-protocol pagp
 channel-group 12 mode desirable
!
interface range fastethernet 0/21 - 22
 channel-protocol lacp
 channel-group 13 mode active

! SW2 — 반드시 desirable 또는 auto (auto만 있으면 SW1이 desirable이므로 OK)
interface range fastethernet 0/23 - 24
 channel-protocol pagp
 channel-group 12 mode desirable

! SW3 — 반드시 active 또는 passive (SW1이 active이므로 passive도 가능)
interface range fastethernet 0/21 - 22
 channel-protocol lacp
 channel-group 13 mode active
```

확인: `SW1# show etherchannel summary`에서 Port-channel 상태가 `SU`(Layer2, In use)로 나오면 정상 번들링된 것이고, 두 스위치가 모두 `auto`(PAgP) 또는 모두 `passive`(LACP)로 설정되어 있으면 포트는 개별 링크로만 남고 Port-channel에 묶이지 않는다.

### EtherChannel Load Balancing 방식 (`IOS XE 16.x+`)

```
! 로드밸런싱 방식 설정 (전역)
SW(config)# port-channel load-balance src-dst-ip     ! 출발지+목적지 IP 해시
SW(config)# port-channel load-balance src-dst-mac    ! 출발지+목적지 MAC (기본)
SW(config)# port-channel load-balance src-dst-port   ! L4 포트 기반 (서버 팜 환경)

! 확인
SW# show etherchannel load-balance
SW# test etherchannel load-balance interface po1 ip 192.168.1.1 192.168.2.1
!  → 위 IP 조합이 어느 멤버 포트로 해시되는지 확인
```

### L3 EtherChannel (`IOS XE 16.x+`)

Phase 2 자료는 L2 EtherChannel 위주. IOS XE L3 Switch에서는 **라우티드 포트 번들** 가능.

```
! L3 EtherChannel 설정 (IOS XE 16.x+)
interface GigabitEthernet1/0/1
 no switchport                   ! L3 포트 전환
 channel-group 1 mode active

interface GigabitEthernet1/0/2
 no switchport
 channel-group 1 mode active

interface Port-channel1
 no switchport                   ! Port-channel도 L3 전환
 ip address 10.1.1.1 255.255.255.252
 no shutdown

! 확인
SW# show etherchannel summary
!  Po1(RU) — R=Layer3, U=in-use
SW# show interface port-channel 1
```

> IOS 15.x Phase 2 자료의 EtherChannel은 L2 번들(Trunk/Access) 위주.  
> IOS XE 16.x L3 Switch에서 L3 EtherChannel은 라우터 링크 집합에 사용.
