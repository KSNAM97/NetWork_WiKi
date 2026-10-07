# Switch 동작원리

스위치는 프레임을 수신할 때 5가지 동작을 수행한다.

---

## Flooding

- 스위치가 **목적지 MAC Address를 모를 때** 수행하는 동작
- 프레임이 들어온 포트를 **제외한 나머지 모든 포트**로 프레임을 전송
- Flooding 대상:
  - **Unknown Unicast Frame** — MAC Table에 없는 유니캐스트
  - **Multicast Frame** — 멀티캐스트 프레임
  - **Broadcast Frame** — 브로드캐스트 프레임

## Learning

- 프레임을 수신했을 때 **출발지 MAC Address를 학습**하는 과정
- Source MAC Address와 수신 포트 번호를 **MAC Address Table에 기록**

## Forwarding

- 목적지 MAC Address를 **알고 있을 때** 해당 포트로만 프레임을 전송
- MAC Address Table에 목적지가 있으면 해당 포트로만 전달

## Filtering

- 스위치가 프레임을 전달하지 않고 **차단**하는 동작
- 출발지와 목적지가 **같은 포트 방향**에 있을 때 발생
- 굳이 다른 포트로 보낼 필요가 없으므로 차단

## Aging

- MAC Address Table에 학습된 정보가 **일정 시간 사용되지 않으면 자동 삭제**
- 장비 이동이나 포트 변경에 대응하기 위한 기능
- **Cisco 스위치 기본 Aging Time: 300초 (5분)**

### 예시: MAC Address Table 학습 과정

**조건**: SW1의 MAC Address Table이 비어있는 상태에서 Fa0/1에 연결된 PC-A가 Fa0/3에 연결된 PC-B에게 처음으로 프레임을 보낸다.

```
! 1) 목적지 MAC을 모르므로 Fa0/1을 제외한 모든 포트로 Flooding
! 2) 동시에 출발지 MAC(PC-A)을 Fa0/1에 Learning
! 3) PC-B가 응답하면 출발지 MAC(PC-B)을 Fa0/3에 Learning
! 4) 이후 PC-A ↔ PC-B 프레임은 서로의 포트로만 Forwarding

SW1# show mac address-table
Vlan    Mac Address       Type        Ports
----    -----------       ----        -----
1       aaaa.aaaa.aaaa    DYNAMIC     Fa0/1
1       bbbb.bbbb.bbbb    DYNAMIC     Fa0/3
```

---

## MAC Address Table 흐름

```
프레임 수신
    │
    ├─ Source MAC 학습 (Learning) → MAC Table에 기록
    │
    └─ Destination MAC 조회
           │
           ├─ Table에 있음 → 해당 포트로 Forwarding
           │                  (출발지 = 목적지 포트? → Filtering)
           │
           └─ Table에 없음 → 전체 포트로 Flooding
```

### 확인 명령어

```
Switch# show mac address-table
```

---

## Hub / Switch / Router 비교

| 장비 | Layer | 도메인 | 주소 | 기반 |
|------|-------|--------|------|------|
| **Hub** | L1 | Collision Domain 공유 | 전기 신호 증폭 | 하드웨어 |
| **Switch** | L2 | Collision Domain 분할, Broadcast Domain 공유 | MAC Address (16진수, 48bit) | 하드웨어 (Cisco IOS 의존) |
| **Router** | L3 | Broadcast Domain 분할 | IP Address (10진수, 32bit) | 소프트웨어 (Cisco IOS 의존) |

- Switch: 프레임 수신 시 Ethernet Header의 **목적지 MAC Address** 기준으로 MAC Address Table 참조 후 Forwarding
- Router: 패킷 수신 시 IP Header의 **목적지 IP Address** 기준으로 Routing Table 참조 후 Forwarding

---

## VLAN (Virtual LAN)

- Switch는 Collision Domain을 분할하지만 **Broadcast Domain을 공유**
- 문제: Host 수 증가 → ARP, DHCP 등 Broadcast Frame 증가
- 해결: VLAN으로 물리적 분할 없이 **논리적 네트워크 분할** 가능
- 장점: Broadcast Traffic 분산, 위치/물리적 제한 없이 연결 가능

### VLAN ID 범위

- VLAN ID: **12bit** 구성 (0 ~ 4095) — 0, 4095는 시스템 예약
- **Standard VLAN**: 1 ~ 1005 (1, 1002, 1003, 1004, 1005는 예약)
- **Extended VLAN**: 1006 ~ 4094 (장비/IOS에 따라 지원 여부 다름)

---

## VLAN 생성 / 수정 / 삭제

```
! VLAN 단일 생성
Switch(config)# vlan 10

! VLAN 복수 생성 (콤마)
Switch(config)# vlan 10,20,30

! VLAN 범위 생성
Switch(config)# vlan 11-30

! VLAN 이름 설정
Switch(config)# vlan 10
Switch(config-vlan)# name CISCO-CCNA

! VLAN 삭제
Switch(config)# no vlan 10
Switch(config)# no vlan 10,20,30
Switch(config)# no vlan 11-30

! VLAN 확인
Switch# show vlan brief
```

### 예시 출력

```
SW1# show vlan brief
VLAN Name                             Status    Ports
---- -------------------------------- --------- -------------------------------
1    default                          active    Fa0/1, Fa0/2 ...
11   CISCO-CCNA                       active
12   CISCO-CCNP                       active
13   LINUX-Server                     active
```

---

## Switchport Mode

### Access Mode

- 하나의 Switchport로 **하나의 VLAN**만 사용
- PC, Server 등 단말 장비 또는 VLAN을 지원하지 않는 장비 연결 포트

```
SW(config)# interface fastethernet 0/1
SW(config-if)# switchport mode access
SW(config-if)# switchport access vlan 10
```

### Trunk Mode

- 하나의 Switchport로 **복수의 VLAN** 사용
- Switch, IP Phone 등 VLAN을 지원하는 장비 연결 포트

### Dynamic Mode (DTP — Dynamic Trunking Protocol)

- **desirable**: DTP Message 송/수신 — 상대가 auto 또는 desirable이면 Trunk 형성
- **auto**: DTP Message 수신만 — 상대가 desirable이어야 Trunk 형성

```
SW(config-if)# switchport mode dynamic desirable
SW(config-if)# switchport mode dynamic auto
```

---

## Access Mode 설정 예시

```
! SW1 — VLAN 10: Fa0/1~4, VLAN 20: Fa0/5~8
vlan 10
 name sol-network
vlan 20
 name sol-system
!
interface fastethernet 0/1
 switchport mode access
 switchport access vlan 10
!
interface fastethernet 0/5
 switchport mode access
 switchport access vlan 20
!
```

---

## Range Command

여러 인터페이스에 동일한 설정을 한 번에 적용

```
! 연속 범위
SW(config)# interface range fa0/1 - 8
SW(config-if-range)# switchport mode access
SW(config-if-range)# switchport access vlan 10

! 비연속 범위 (콤마)
SW(config)# interface range fa0/1, fa0/3, fa0/5, fa0/7
SW(config-if-range)# switchport mode access
SW(config-if-range)# switchport access vlan 30

! 혼합 범위
SW(config)# interface range fa0/1-4, fa0/9-12
SW(config-if-range)# switchport mode access
SW(config-if-range)# switchport access vlan 50
```

---

## Trunk

Switch간 연결에서 Access Mode 사용 시 VLAN 수만큼 링크가 필요하지만, Trunk Mode 사용 시 **하나의 링크로 다수의 VLAN 통신** 가능

### Trunk 설정

```
! IEEE 802.1Q (표준)
interface fastethernet 0/24
 switchport trunk encapsulation dot1q
 switchport mode trunk

! ISL (Cisco 전용)
interface fastethernet 0/24
 switchport trunk encapsulation isl
 switchport mode trunk

! 2950 이하 (ISL 미지원 — dot1q 자동 사용)
interface fastethernet 0/24
 switchport mode trunk

! 확인
SW# show interface trunk
```

### Dot1q vs ISL

| 항목 | IEEE 802.1Q | ISL |
|------|-------------|-----|
| 표준 | IEEE 국제 표준 | Cisco 전용 |
| Native VLAN | 지원 (태그 없는 프레임 → Native VLAN 처리) | 미지원 (태그 없으면 Drop) |
| 기본 Native VLAN | VLAN 1 | — |

### IEEE 802.1Q Tagging (4Byte 확장)

```
Access 모드 Ethernet Header:
| Destination MAC | Source MAC | Type | Data |

Trunk 모드 Ethernet Header (802.1Q):
| Destination MAC | Source MAC | Tag(4Byte) | Type | Data |

Tag 구조 (32bit):
- EtherType (16bit): 0x8100 — 802.1Q 식별
- Priority (3bit):   QoS 우선순위 (8단계)
- CFI (1bit):        0=Ethernet, 1=Token-ring
- VLAN-ID (12bit):   VLAN 정보 (0~4095), 태그 없으면 Native VLAN 처리
```

---

## Trunk Allowed VLAN

Trunk 포트가 기본적으로 모든 VLAN을 포워딩하지만, 특정 VLAN만 허용하거나 제외 가능

```
! 기본 (전체 허용)
switchport trunk allowed vlan all

! 특정 VLAN만 허용
switchport trunk allowed vlan 1,10,20,30

! VLAN 추가
switchport trunk allowed vlan add 40

! VLAN 제거
switchport trunk allowed vlan remove 10

! 특정 VLAN 제외 (나머지 전체)
switchport trunk allowed vlan except 100

! 확인
SW# show interface trunk
```

---

## Router-on-a-Stick (Sub-Interface)

물리적 인터페이스 하나를 여러 논리적 Sub-Interface로 분할하여 VLAN 간 라우팅 구현

```
! R1 설정
interface fastethernet 0/0
 no shutdown
!
interface fastethernet 0/0.10           ! Sub-interface 생성
 encapsulation dot1q 10                ! VLAN 10 태깅
 ip address 192.168.10.254 255.255.255.0
!
interface fastethernet 0/0.20
 encapsulation dot1q 20
 ip address 192.168.20.254 255.255.255.0
!
interface fastethernet 0/0.30
 encapsulation dot1q 30
 ip address 192.168.30.254 255.255.255.0
!

! SW1 설정 — R1 연결 포트를 Trunk로
interface fastethernet 0/24
 switchport trunk encapsulation dot1q
 switchport mode trunk
!
```

```
R1# show ip route
C    192.168.10.0/24 is directly connected, FastEthernet0/0.10
C    192.168.20.0/24 is directly connected, FastEthernet0/0.20
C    192.168.30.0/24 is directly connected, FastEthernet0/0.30
```

---

## VTP (VLAN Trunking Protocol)

Cisco 전용 프로토콜 — 여러 Switch 간 VLAN Database를 자동 동기화

### VTP 동작 요건

- Switch간 Trunk 연결 필요
- VTP Domain 이름 일치 (Trunk 연결 시 자동 학습)
- VTP Password 일치 (설정한 경우)

### VTP Mode

| Mode | VLAN 생성/수정/삭제 | 동기화 전파 | 동기화 수신 |
|------|-------------------|------------|------------|
| **Server** | O | O | O |
| **Client** | X | O | O |
| **Transparent** | O (로컬만) | X | O (통과 전달) |

> Transparent Mode는 Standard VLAN + Extended VLAN 모두 생성 가능

### VTP 설정

```
SW(config)# vtp mode server
SW(config)# vtp mode client
SW(config)# vtp mode transparent

SW(config)# vtp domain soldesk
SW(config)# vtp password sol1234

! 확인
SW# show vtp status
```

### VTP 메시지 유형

| 메시지 | 설명 |
|--------|------|
| **Summary-Advertise** | VTP Server가 VLAN DB 변경 시 Revision Number와 함께 주기 전송 |
| **Advertise-Request** | Revision Number가 자신보다 높으면 전체 DB 요청 |
| **Subset-Advertise** | Advertise-Request에 응답하여 전체 VLAN DB 전송 |

---

## STP (Spanning-Tree Protocol) — IEEE 802.1D / PVST

여러 Switch 경로에서 **Loop 방지**를 위한 프로토콜

### Loop 발생 문제

- **Broadcast Storm**: 브로드캐스트 프레임이 무한 순환
- **중복 프레임**: 동일 프레임이 여러 경로로 복수 수신
- **MAC Table 불안정**: 동일 MAC이 여러 포트에서 반복 학습

### BPDU (Bridge Protocol Data Unit)

- Bridge-ID (VLAN Priority + MAC Address)
- Cost (10M=100, 100M=19, 1G=4, 10G=2)
- Port-Priority (기본 128.x, x=포트 번호)
- Hello-time: 2초, Max-age: 20초, Forward-delay: 15초

### Root Bridge 선출

1. **Priority 낮은** Switch (기본 32768 + VLAN 번호)
2. Priority 동일 시 **MAC Address 낮은** Switch

### Port 역할

| 역할 | 설명 |
|------|------|
| **DP (Designated Port)** | BPDU 송신 포트 — Root Bridge의 모든 포트 + 세그먼트별 최선 포트 |
| **RP (Root Port)** | Root Bridge 방향으로 BPDU를 수신하는 포트 (Root Bridge까지 가장 좋은 경로) |
| **AP (Alternate Port)** | Block 상태 — Cost/Priority/Bridge-ID 열세 포트 |

### STP 포트 상태 전이

| 상태 | BPDU | MAC 학습 | 데이터 전달 | 지속 시간 |
|------|------|----------|------------|-----------|
| Disable | X | X | X | — |
| Blocking | O (수신만) | X | X | Max-age 20초 |
| Listening | O | X | X | Forward-delay 15초 |
| Learning | O | O | X | Forward-delay 15초 |
| Forwarding | O | O | O | 정상 운영 |

> Root Bridge 재선출 시 (Non-Root → Root): Blocking → Listening(15s) → Learning(15s) → Forwarding = **총 30초**  
> Root Bridge 완전 장애 복구 시: Blocking(Max-age 20s) → Listening(15s) → Learning(15s) → Forwarding = **총 50초**

### STP 설정 (Priority로 Root Bridge 지정)

**예시**: 네트워크 설계상 SW2를 VLAN 10, 20, 30, 40의 Root Bridge로 지정해야 하는 경우, priority를 4096(가장 낮은 배수 값)으로 낮춰 강제로 선출되도록 한다.

```
! VLAN별 Root Bridge 지정 — priority는 4096의 배수
SW2(config)# spanning-tree vlan 10,20,30,40 priority 4096

! Cost 조정으로 Block Port 제어 (수신측에서 설정)
interface fastethernet 0/23
 spanning-tree vlan 30,40 cost 100

! Port-Priority 조정 (Root Bridge에서 설정)
interface fastethernet 0/23
 spanning-tree vlan 30,40 port-priority 192

! STP 타이머 조정
spanning-tree vlan 1 hello-time 2
spanning-tree vlan 1 forward-time 15
spanning-tree vlan 1 max-age 20

! 확인
SW# show spanning-tree vlan 10
SW# debug spanning-tree events
```

---

## IOS XE 16.x / 17.x 최신 트렌드

### MAC Address Table 개선 (`IOS XE 16.x+`)

```
! MAC Address Table 확인 (IOS XE — 기존과 동일하나 출력 포맷 변경)
Switch# show mac address-table
Switch# show mac address-table dynamic
Switch# show mac address-table count

! Aging Time 조정 (기본 300초)
Switch(config)# mac address-table aging-time 600

! Static MAC Entry 등록 (특정 포트 고정)
Switch(config)# mac address-table static 0011.2233.4455 vlan 10 interface GigabitEthernet1/0/1
```

### VLAN 관련 IOS XE 변경사항

```
! IOS XE 16.x — Extended VLAN(1006~4094) VTP 없이도 설정 가능 (Transparent Mode 불필요)
Switch(config)# vlan 2000
Switch(config-vlan)# name EXTENDED_VLAN

! IOS XE 16.x — VLAN 상세 확인
Switch# show vlan id 10
Switch# show vlan brief
```

### PVST+ → Rapid-PVST+ 전환 (`IOS XE 16.x 기본`)

IOS XE 16.x 이상의 Catalyst 스위치는 기본 STP 모드가 **Rapid-PVST+**로 변경됨 (IOS 15.x까지는 PVST+)

```
! 현재 STP 모드 확인
Switch# show spanning-tree summary

! Rapid-PVST+ 명시 설정 (IOS XE 기본값)
Switch(config)# spanning-tree mode rapid-pvst

! MSTP (Multiple Spanning Tree) — 대규모 환경 권장
Switch(config)# spanning-tree mode mst
Switch(config)# spanning-tree mst configuration
Switch(config-mst)# name REGION_A
Switch(config-mst)# revision 1
Switch(config-mst)# instance 1 vlan 10,20
Switch(config-mst)# instance 2 vlan 30,40
```

> Rapid-PVST+는 기존 PVST+와 달리 Listening 단계 없이 **Blocking → Learning → Forwarding** 전이가 1-2초 내 완료 (PortFast 없이도 빠른 수렴)
