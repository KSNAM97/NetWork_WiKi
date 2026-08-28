# Frame Relay

- WAN 구간에서 사용되는 패킷 교환 기술
- **DLCI (Data Link Connection Identifier)**: Frame Relay에서 VC(Virtual Circuit)를 식별하는 번호
- **LMI (Local Management Interface)**: Router와 Frame Relay 스위치 간 상태 정보 교환 프로토콜

### 장단점

- **장점**: 전용회선과 유사한 전송 보장 서비스를 더 저렴한 비용으로 제공, Layer 2 계층 서비스이므로 고속 서비스 구현 가능, 하나의 인터페이스로 여러 지사(지점)와 동시 연결(Multi-access)이 가능
- **단점**: 다수의 사용자가 회선을 공유하므로 보안에 취약하고, 동시 통신 시 신뢰적인 대역폭 보장이 어려움

---

## 주요 개념

| 용어 | 설명 |
|------|------|
| DLCI | 논리적 회선 식별자 (로컬 의미) |
| PVC | Permanent Virtual Circuit — 고정된 가상 회선 |
| SVC | Switched Virtual Circuit — 동적 가상 회선 |
| LMI | Router ↔ FR 스위치 간 상태 교환 |
| FECN | Forward Explicit Congestion Notification |
| BECN | Backward Explicit Congestion Notification |

---

## LMI 타입

| LMI 타입 | 설명 |
|----------|------|
| cisco | CISCO 전용 (기본값) |
| ansi | ANSI T1.617 Annex D |
| q933a | ITU-T Q.933 Annex A |

```
R(config-if)# frame-relay lmi-type [cisco|ansi|q933a]
```

### PVC 상태

| 상태 | 설명 |
|------|------|
| Active | PVC 정상 동작 |
| Inactive | 로컬 측은 정상이나 원격 측 문제 |
| Deleted | DLCI가 스위치에 설정되지 않음 |

---

## FRSW (Frame Relay Switch) 설정

Frame Relay 스위치 역할을 수행하는 라우터 설정

```
FRSW(config)# frame-relay switching            ! FR 스위칭 기능 활성화

FRSW(config)# interface serial 1/0
FRSW(config-if)# encapsulation frame-relay
FRSW(config-if)# frame-relay intf-type dce     ! DCE 역할 (클록 제공)
FRSW(config-if)# clock rate 64000
FRSW(config-if)# no shutdown
FRSW(config-if)# frame-relay route 102 interface serial 1/1 201   ! DLCI 102 → Serial1/1의 DLCI 201로 전달

FRSW(config)# interface serial 1/1
FRSW(config-if)# encapsulation frame-relay
FRSW(config-if)# frame-relay intf-type dce
FRSW(config-if)# clock rate 64000
FRSW(config-if)# no shutdown
FRSW(config-if)# frame-relay route 201 interface serial 1/0 102   ! DLCI 201 → Serial1/0의 DLCI 102로 전달
```

---

## 주 Interface (Main Interface) 설정

```
R1(config)# interface serial 1/0
R1(config-if)# ip address 10.1.12.1 255.255.255.0
R1(config-if)# encapsulation frame-relay
R1(config-if)# no shutdown
R1(config-if)# frame-relay map ip 10.1.12.2 102 broadcast   ! 목적지 IP → DLCI 102 매핑, broadcast: 멀티캐스트/브로드캐스트 허용
```

---

## Sub-interface 방식

### Point-to-Point (P2P)

- 각 Sub-interface가 하나의 PVC에 대응
- Split-horizon 문제 없음
- IP 주소 블록 낭비 있음

```
R1(config)# interface serial 1/0
R1(config-if)# no shutdown
R1(config-if)# encapsulation frame-relay
!
R1(config)# interface serial 1/0.123 point-to-point
R1(config-subif)# ip address 10.1.12.1 255.255.255.252
R1(config-subif)# frame-relay interface-dlci 102
```

### Multipoint

- 하나의 Sub-interface에 복수 PVC 연결
- Split-horizon 문제 발생 가능
- IP 주소 절약 가능

```
R1(config)# interface serial 1/0.123 multipoint
R1(config-subif)# ip address 10.1.123.1 255.255.255.0
R1(config-subif)# frame-relay map ip 10.1.123.2 102 broadcast
R1(config-subif)# frame-relay map ip 10.1.123.3 103 broadcast
```

### 예제: 한 인터페이스에서 Multipoint + P2P 혼합 구성

**조건**: R2는 물리 인터페이스 Serial 1/0 하나로 두 종류의 PVC를 동시에 운용해야 한다. R1·R3와는 192.168.100.0/24 대역의 Multipoint 서브인터페이스(DLCI 201 → R1, DLCI 203 → R3)로 묶고, R4와는 192.168.24.0/24 대역의 별도 P2P 서브인터페이스(DLCI 204)로 연결한다.

```
R2(config)# interface serial 1/0
R2(config-if)# no shutdown
R2(config-if)# encapsulation frame-relay
!
R2(config)# interface serial 1/0.123 multipoint
R2(config-subif)# ip address 192.168.100.2 255.255.255.0
R2(config-subif)# frame-relay map ip 192.168.100.1 201
R2(config-subif)# frame-relay map ip 192.168.100.3 203
!
R2(config)# interface serial 1/0.24 point-to-point
R2(config-subif)# ip address 192.168.24.2 255.255.255.0
R2(config-subif)# frame-relay interface-dlci 204
```

`R2# show ip route`로 확인하면 두 서브인터페이스가 각각 Connected로 등록된다.
```
C    192.168.24.0/24 is directly connected, Serial1/0.24
C    192.168.100.0/24 is directly connected, Serial1/0.123
```

---

## Multipoint의 두 가지 구성 — Full-mesh vs Hub-and-Spoke

Frame Relay Multipoint는 하나의 (Sub-)Interface로 여러 장비를 동일 네트워크(NBMA)에 연결하는 방식이며, 연결 구조에 따라 두 가지로 나뉜다.

| 구성 | 설명 | Split-horizon 문제 |
|------|------|---------------------|
| **Full-mesh** | Frame Relay로 연결된 모든 라우터 간에 V/C를 전부 연결 (R1-R2, R2-R3, R1-R3 모두 직접 PVC) | 발생하지 않음 — 모든 라우터가 서로 직접 연결되어 있음 |
| **Hub-and-Spoke** | 중앙(Hub) 라우터 하나만 모든 Spoke와 PVC로 연결, Spoke끼리는 PVC 없이 Hub를 경유 | **발생** — 아래 참조 |

```
! Full-mesh 예시: R1, R2, R3가 서로 DLCI로 직접 연결된 192.168.100.0/24 Multipoint 구성
! R1
interface serial 1/0.123 multipoint
 ip address 192.168.100.1 255.255.255.0
 frame-relay map ip 192.168.100.2 102
 frame-relay map ip 192.168.100.3 103
```

---

## Split-horizon 문제

Hub-and-Spoke 구조에서 Multipoint 구성 시 발생
- Hub가 Spoke1에서 받은 라우팅 정보를 같은 Interface의 Spoke2로 전달 불가
- 물리적으로는 Hub와 모든 Spoke가 통신 가능하지만, Spoke1과 Spoke2 사이의 경로 정보가 Hub를 거쳐 전파되지 못해 Spoke간 라우팅 학습이 실패하는 문제

**해결책**
```
R(config-if)# no ip split-horizon eigrp 100
```

---

## 정보 확인

```
R# show frame-relay pvc          ! PVC 상태 및 DLCI 확인
R# show frame-relay map          ! IP ↔ DLCI 매핑 테이블 확인
R# show frame-relay lmi          ! LMI 타입 및 통계 확인
R# show frame-relay route        ! FRSW에서 DLCI 스위칭 경로 확인
R# show interface serial 1/0     ! 인터페이스 상태 및 인캡슐레이션 확인
```

`show frame-relay route` 출력 예시 (FRSW):
```
Input Intf    Input DLCI    Output Intf    Output DLCI    Status
Serial1/0     102           Serial1/1      201            active
Serial1/1     201           Serial1/0      102            active
```

`show frame-relay map` 출력 예시:
```
Serial1/0.12 (up): point-to-point dlci, dlci 102(0x66,0x1860), broadcast,
              status defined, active
```

---

## IOS XE 16.x 관점

IOS XE 15.2T 이후 Frame Relay는 deprecated 되었으며, MPLS/VPLS로 대체됨.

- IOS XE 16.x 이상에서는 Frame Relay 기능이 지원되지 않는 플랫폼 증가
- 현대 WAN 환경에서는 MPLS L3VPN, VPLS(L2VPN), SD-WAN으로 대체
- 레거시 환경 유지보수 목적으로만 Frame Relay 지식 활용
