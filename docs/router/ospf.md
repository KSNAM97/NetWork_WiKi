# OSPF (Open Shortest Path First)

- 표준 Link State 라우팅 프로토콜 (RFC 2328)
- Protocol 번호: **89**
- AD (Administrative Distance): **110**
- Metric: **Cost** (= 100Mbps / Interface Bandwidth)
- 알고리즘: **Dijkstra (SPF)**

---

## 사전 설정 (Pre-config)

OSPF 실습을 위한 기본 토폴로지 — R1~R5 5대의 라우터로 구성되며, R1-R2-R3는 Frame-Relay 멀티포인트(13.13.9.0/24, Area 0)로 연결되고 R1-R3는 별도 P2P 서브인터페이스(13.13.10.0/24)로도 연결된다. 모든 LAN 인터페이스는 `bandwidth 100000`(100Mbps)으로 통일해 Cost 계산 기준을 맞춘다.

- R1: Lo0 13.13.1.1/24, Fa0/0 150.1.13.1/24(R4 방향), Fa0/1 13.13.11.1/24, S1/0.123(F-R Multipoint) 13.13.9.1/24, S1/1(F-R) 13.13.10.1/24(R3 방향)
- R2: Lo0 13.13.2.2/24, Fa0/1 13.13.12.2/24, S1/0.123(F-R Multipoint, description "OSPF Area 0") 13.13.9.2/24
- R3: Lo0 13.13.3.3/24, Fa0/0 150.3.13.3/24(R5 방향), Fa0/1 13.13.13.3/24, S1/0.123(F-R Multipoint) 13.13.9.3/24, S1/1(F-R) 13.13.10.3/24(R1 방향)
- R4: Lo0 13.13.4.4/24, Fa0/0 150.1.13.4/24(R1 방향), Fa0/1 13.13.14.4/24 — 별도 Area(예: Area 14)로 R1에 연결되는 Stub Area 실습용 라우터
- R5: Lo0 13.13.5.5/24, Fa0/0 150.3.13.5/24(R3 방향), Fa0/1 13.13.15.5/24

공통 초기화(`no ip domain-lookup`, `enable secret cisco`, `line con/vty password cisco`)는 모든 라우터에 적용된다.

```
! R1
hostname R1
!
interface loopback 0
 ip add 13.13.1.1 255.255.255.0
!
interface fastethernet0/0
 ip add 150.1.13.1 255.255.255.0
 no shutdown
 bandwidth 100000
!
interface fastethernet0/1
 ip add 13.13.11.1 255.255.255.0
 no shutdown
 bandwidth 100000
!
interface serial 1/0
 no shutdown
 encapsulation frame-relay
 no frame-relay inverse-arp
!
interface serial 1/0.123 multipoint
 frame-relay map ip 13.13.9.2 102 broadcast
 frame-relay map ip 13.13.9.3 102 broadcast
 ip add 13.13.9.1 255.255.255.0
!
interface serial 1/1
 no shutdown
 encapsulation frame-relay
 no frame-relay inverse-arp
 ip add 13.13.10.1 255.255.255.0
 frame-relay map ip 13.13.10.3 113 broadcast

! R2 — Area 0 서브인터페이스 표시
interface serial1/0.123 multipoint
 description ### OSPF Area 0 ###
 ip add 13.13.9.2 255.255.255.0
 frame-relay map ip 13.13.9.1 201 broadcast
 frame-relay map ip 13.13.9.3 203 broadcast

! R4 — 인접 Area(예: Area 14) 실습용, ASBR/Stub 시나리오에 사용
interface fastethernet0/0
 ip add 150.1.13.4 255.255.255.0
 no shutdown
 bandwidth 100000
!
interface fastethernet0/1
 ip address 13.13.14.4 255.255.255.0
 no shutdown
 bandwidth 100000
```

---

## OSPF Area

- OSPF는 Area 단위로 네트워크를 분할
- **Area 0 (Backbone Area)**: 모든 Area가 반드시 Area 0에 연결되어야 함
- ABR (Area Border Router): 두 Area에 걸쳐 있는 라우터

---

## DR / BDR (Designated Router / Backup DR)

Multi-access 네트워크(Ethernet)에서 OSPF 인접성 최적화를 위해 선출

| 역할 | 설명 |
|------|------|
| DR | 모든 OSPF 정보를 수집 후 다른 라우터에 배포 |
| BDR | DR 장애 시 즉시 DR 역할 승계 |
| DROther | DR/BDR이 아닌 나머지 라우터 |

### DR 선출 기준 (우선순위 높은 순)

1. **OSPF Priority** (기본 1, 높을수록 유리, 0=선출 제외)
2. **Router-ID** (높을수록 유리)

### Router-ID 결정 순서

1. `router-id` 명령으로 수동 설정 (최우선)
2. Loopback Interface 중 가장 높은 IP
3. 활성화된 Interface 중 가장 높은 IP

---

## 기본 설정

```
Router(config)# router ospf [Process-ID]                     ! Process-ID는 로컬 식별자 (이웃 간 달라도 됨)
Router(config-router)# router-id [Router-ID]                 ! 수동 지정 시 재시작 없이 즉시 적용 (권장)
Router(config-router)# passive-interface default             ! 모든 인터페이스에서 Hello 비활성화 (보안)
Router(config-router)# no passive-interface [interface]      ! 이웃을 맺을 인터페이스만 개별 활성화
Router(config-router)# network [IP] [Wildcard] area [Area]  ! 해당 IP 범위의 인터페이스를 지정 Area에 포함
```

### 설정 예시

```
R1(config)# router ospf 1
R1(config-router)# router-id 1.1.1.1                ! 루프백 기반 고정 ID 권장 — 재부팅 후에도 유지
R1(config-router)# passive-interface default         ! LAN 포트 등 불필요한 인터페이스 보호
R1(config-router)# no passive-interface serial 1/0          ! WAN 연결 — OSPF 이웃 허용
R1(config-router)# no passive-interface fastethernet 0/0    ! 다른 라우터 연결 포트 허용
R1(config-router)# network 13.13.1.0 0.0.0.255 area 0      ! Wildcard는 서브넷 마스크와 반전값
R1(config-router)# network 13.13.9.0 0.0.0.255 area 0
R1(config-router)# network 150.1.13.0 0.0.0.255 area 0
```

---

## OSPF 5가지 PDU

| PDU | 설명 |
|-----|------|
| **Hello** | 이웃 탐색 및 유지 — Neighbor Table 등록, 주기적 교환 |
| **DBD (Database Description)** | LSDB 요약 교환 — 이웃 간 보유 LSA 목록 비교 |
| **LSR (Link-State Request)** | DBD 비교 후 자신에게 없는 LSA 요청 |
| **LSU (Link-State Update)** | LSR 응답 — 요청받은 LSA 전송 |
| **LSAck (Link-State Acknowledge)** | DBD/LSR/LSU 수신 확인 응답 |

### Hello Packet 필드 (* = 양쪽 일치 필수)

- Area-ID `*`, Hello/Dead Interval `*`, SubnetMask `*`, MTU `*`, Authentication `*`, Stub `*`
- Router-ID, OSPF Priority, DR/BDR (참조용)

---

## Network Type별 Hello/Dead 타이머 및 DR/BDR

| Network Type | Hello/Dead | DR/BDR | 대표 인터페이스 |
|-------------|-----------|--------|----------------|
| Broadcast (BMA) | 10초 / 40초 | 선출 | Ethernet, FastEthernet |
| Non-Broadcast (NBMA) | 30초 / 120초 | 선출 | Frame-Relay Multipoint |
| Point-to-Point | 10초 / 40초 | 없음 | PPP, HDLC, Frame-Relay P2P |
| Point-to-Multipoint | 30초 / 120초 | 없음 | OSPF가 자동 설정하는 환경 |

---

## OSPF Area 구조

### Single Area

- 네트워크 전체를 하나의 Area로 구성
- Area 번호는 0 ~ 4294967295 중 하나 사용

### Multiple Area

- 두 개 이상의 Area를 사용하는 구조
- 모든 Area는 **Backbone Area(Area 0)** 에 반드시 연결
- 동일 Area 내에서만 LSDB 동기화
- Area 간에는 요약 경로만 교환 → LSA Flooding 범위 최소화

| 라우터 역할 | 설명 |
|------------|------|
| **ABR** (Area Border Router) | 두 Area 경계에 위치 — 각 Area의 LSDB를 동기화 |
| **ASBR** (AS Boundary Router) | 외부 프로토콜(BGP, EIGRP 등)의 경로를 OSPF로 재분배 |

---

## 등가 부하 분산 (Equal-Cost Load Balancing)

- 동일 목적지에 Cost가 같은 경로 여러 개 존재 시 분산
- 기본 최대 경로: **4개** (IOS 12.2/12.4: 최대 16개)

```
router ospf 1
 maximum-paths 8
```

---

## NBMA 구간 Network Type

OSPF는 인터페이스의 Network Type에 따라 Hello/Dead 타이머와 DR/BDR 선출 여부가 결정됨

| Network Type | Hello/Dead | DR/BDR | 대표 환경 |
|--------------|-----------|--------|---------|
| Broadcast (BMA) | 10/40 | 선출함 | Ethernet, FastEthernet |
| Non Broadcast (NBMA) | 30/120 | 선출함 | F/R Multipoint, NBMA 구간 |
| Point-to-Point | 10/40 | 선출 안함 | PPP, HDLC, F/R P2P |
| Point-to-Multipoint | 30/120 | 선출 안함 | OSPF 전용 (NBMA를 논리적 P2P로 변환) |

### NBMA 구간 OSPF 인접성 연결 방법

1. DR/BDR 선출 시: `neighbor` 명령으로 인접 라우터 지정
2. 두 대 라우터, DR/BDR 없이: Network type을 `point-to-point`로 변경
3. 세 대 이상, DR/BDR 없이: Network type을 `point-to-multipoint`로 변경

### Point-to-Multipoint 설정 (DR/BDR 선출 안함)

```
# R1, R2, R3 — Frame-Relay 구간 Sub-interface에 적용
interface serial 1/0.123
 ip ospf network point-to-multipoint
```

> Point-to-Multipoint 설정 시 라우팅 테이블에서 각 Spoke IP가 /32로 표시됨

### NBMA에서 neighbor 명령 사용 (DR/BDR 선출)

Hub 라우터가 DR 역할을 하고, Spoke에서는 Priority를 0으로 설정

```
# Hub (DR 역할)
interface serial 1/0.123
 ip ospf priority 255
router ospf 1
 neighbor 13.13.9.1    ! Spoke1 IP
 neighbor 13.13.9.3    ! Spoke3 IP

# Spoke (DR/BDR 선출 제외)
interface serial 1/0.123
 ip ospf priority 0
```

**예시 문제**: R1, R2, R3이 Frame-Relay(NBMA) 구간으로 연결되어 있고, R2가 반드시 DR로 선출되어야 하며 BDR은 선출하지 않아야 한다는 조건이 주어졌다면 어떻게 설정할까? NBMA는 기본적으로 Hello/Dead가 30/120초이고 `network` 명령만으로는 인접성이 아예 맺어지지 않는다.

```
! R2 (Hub, DR로 강제)
interface serial 1/0.123
 ip ospf priority 255        ! Priority 가장 높게 — DR 선출 보장
!
router ospf 1
 neighbor 13.13.9.1          ! NBMA에서는 neighbor 명령으로 지정한 라우터와만 인접성 형성
 neighbor 13.13.9.3

! R1, R3 (Spoke, DR/BDR 선출 제외)
interface serial 1/0.123
 ip ospf priority 0          ! Priority 0 = DR/BDR 선출 대상에서 제외
```

확인: `R2# show ip ospf neighbor` 에서 R1/R3이 `FULL/DROTHER`로, `R1# show ip ospf neighbor` 에서 R2가 `FULL/DR`로 보이면 조건 충족. BDR 자리는 Priority 0인 Spoke들만 있으므로 자동으로 선출되지 않는다.

### Loopback Interface Network Type 변경

Loopback Interface는 기본적으로 LOOPBACK 타입으로 동작하여 /32로 광고됨.
원래 Subnet Mask로 광고하려면 Network Type을 변경해야 함.

```
interface loopback 0
 ip ospf network point-to-point    ! /24 등 실제 마스크로 광고
```

---

## OSPF Cost 조정

```
Router(config-if)# ip ospf cost [값]
Router(config-if)# bandwidth [kbps]
```

### Cost = 100Mbps / Interface Bandwidth

| Interface | 기본 Cost |
|-----------|-----------|
| FastEthernet (100Mbps) | 1 |
| Ethernet (10Mbps) | 10 |
| T1 (1.544Mbps) | 64 |
| Serial (64kbps) | 1562 |

### 기준 대역폭 변경 (Reference Bandwidth)

기본 기준 대역폭은 100Mbps. GigabitEthernet 환경에서는 GE(1Gbps)와 FE(100Mbps)가 동일한 Cost 1로 계산됨.
OSPF 도메인 전체에서 동일하게 변경해야 함.

```
router ospf 1
 auto-cost reference-bandwidth 1000    ! 1000Mbps(1Gbps) 기준으로 변경
```

| Interface | 기준 100Mbps 시 Cost | 기준 1000Mbps 시 Cost |
|-----------|---------------------|----------------------|
| GigabitEthernet | 1 (부정확) | 1 |
| FastEthernet | 1 | 10 |
| Ethernet | 10 | 100 |
| Serial (1.544Mbps) | 64 | 647 |

---

## LSA 타입 (Link-State Advertisement)

OSPF는 Area 단위로 LSDB(Link-State Database)를 관리하며 LSA 타입별로 정보를 구분

| LSA 타입 | 생성 | 용도 | 전파 범위 | 라우팅 테이블 표기 |
|----------|------|------|----------|-----------------|
| Type-1 (Router) | 모든 OSPF 라우터 | 동일 Area 내 링크 정보 | 동일 Area | O |
| Type-2 (Network) | DR | DR의 Multi-access 네트워크 정보 | DR이 포함된 네트워크 | 표기 없음 |
| Type-3 (Summary Net) | ABR | 다른 Area 간 업데이트 | ABR이 포함된 Area | O IA |
| Type-4 (Summary ASBR) | ABR | ASBR의 위치 정보 전파 | ABR이 포함된 Area | 표기 없음 |
| Type-5 (External) | ASBR | 외부 프로토콜 재분배 정보 | OSPF 전체 도메인 | O E1, O E2 |
| Type-7 (NSSA External) | NSSA 내 ASBR | NSSA 구간 재분배 (LSA-5 대체) | NSSA Area | O N1, O N2 |

> O E1: 누적 Metric (ASBR까지의 경로 복수개 환경에서 사용)  
> O E2: 고정 Metric (ASBR까지의 경로 단일 환경에서 사용, 기본값)

```
! ASBR에서 재분배 시 metric-type 지정
router ospf 1
 redistribute eigrp 100 metric-type 1 subnets    ! E1: 누적 Metric
 redistribute eigrp 100 metric-type 2 subnets    ! E2: 고정 Metric (기본)
```

### OSPF Database 조회

```
R# show ip ospf database              ! LSDB 전체 목록
R# show ip ospf database router       ! LSA Type-1 확인
R# show ip ospf database network      ! LSA Type-2 확인
R# show ip ospf database summary      ! LSA Type-3 확인
R# show ip ospf database asbr-summary ! LSA Type-4 확인
R# show ip ospf database external     ! LSA Type-5 확인
R# show ip ospf border-router         ! ABR/ASBR 확인
```

---

## Stub Area

특정 Area에서 LSA를 차단하여 라우팅 테이블을 간소화하는 기능

| 종류 | LSA 차단 | Default-route | 특징 |
|------|----------|---------------|------|
| Stub | ABR이 LSA-4, 5 차단 | 자동 생성 | ASBR이 포함된 Area는 사용 불가 |
| Totally Stub | ABR이 LSA-3, 4, 5 차단 | 자동 생성 | ASBR이 포함된 Area는 사용 불가 |
| NSSA | ABR이 LSA-4, 5 차단 | 수동 생성 필요 | ASBR이 포함되어도 사용 가능, Type-7로 재분배 |
| Totally NSSA | ABR이 LSA-3, 4, 5 차단 | 자동 생성 | NSSA + Totally Stub 결합 |

### Stub Area 설정 조건

- Backbone Area (Area 0)는 Stub 불가
- Area 내 모든 라우터에서 공통 설정 필요 (인접성 조건)
- Transit Area (Virtual-link 존재 Area)는 Stub 불가
- ASBR이 포함된 Area는 Stub 불가 (NSSA는 예외)

### Stub 설정

```
# ABR + 해당 Area의 모든 라우터에서 설정
router ospf 1
 area 14 stub
```

### Totally Stub 설정

```
# ABR에서만 no-summary 추가
router ospf 1
 area 14 stub no-summary    ! ABR

# Spoke 라우터
router ospf 1
 area 14 stub               ! no-summary 없이 설정
```

### NSSA 설정

```
# ABR — default-information-originate로 수동 Default-route 생성
router ospf 1
 area 14 nssa default-information-originate

# ASBR (NSSA Area 내 재분배 라우터)
router ospf 1
 area 14 nssa
```

**예시 문제**: Area 14에 속한 R4가, 같은 OSPF 내부 경로(O, O IA)는 라우팅 테이블 상세 정보로 통신해야 하지만 EIGRP에서 재분배되어 들어온 외부 경로(O E1/O E2)는 라우팅 테이블에서 아예 보이지 않아야 한다(대신 Default-route로만 통신). 단 Area 0에서는 재분배 정보가 그대로 보여야 한다면, Area 14를 어떤 종류의 Stub으로 만들어야 할까? — ASBR이 다른 Area(0)에 있고 Area 14 자체에는 없으므로 일반 **Stub**(Totally Stub 아님, NSSA 아님)이 조건에 맞는다.

```
! Area 14에 속한 모든 라우터(ABR인 R1 포함, R4 포함)에서 동일하게 설정 — Stub는 인접성 조건이므로 전원 일치 필수
router ospf 1
 area 14 stub
```

확인: `R4# show ip route ospf` 에서 Stub 적용 전에는 `O E1 13.13.5.0 ...` 같은 재분배 경로가 보이지만, 적용 후에는 O/O IA만 남고 `O*IA 0.0.0.0/0 ...` (Default-route)이 대신 나타난다. 만약 ASBR이 Area 14 안에 있었다면 이 조건 자체가 성립할 수 없다(ASBR 포함 Area는 Stub 불가, NSSA만 가능).

### Totally NSSA 설정

```
# ABR
router ospf 1
 area 14 nssa no-summary    ! Default-route 자동 생성

# ASBR
router ospf 1
 area 14 nssa
```

---

## OSPF 경로 요약

### ABR에서 Internal 경로 요약

Area 경계에서 Inter-Area 정보 축약

```
router ospf 1
 area 14 range 100.100.0.0 255.255.248.0    ! Area 14의 /24 8개를 하나의 /21로 요약
```

### ASBR에서 External 경로 요약

재분배된 외부 경로 축약

```
router ospf 1
 summary-address 200.200.8.0 255.255.248.0  ! 재분배 시 /24 8개를 /21로 요약
```

---

## OSPF 인증

```
Router(config-if)# ip ospf authentication
Router(config-if)# ip ospf authentication-key [비밀번호]
```

---

## IOS XE 최신 트렌드 — 고급 기능

> 출처: [Cisco IOS XE 17.x OSPF Configuration Guide](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_iro-cfg-0.html)

### Fast Hello (서브초 수렴)

일반 Hello 10초 → Dead 40초 대신, 200ms 간격으로 헬로를 전송하여 링크 장애를 1초 이내 감지

```
interface GigabitEthernet 0/1
 ! dead-interval을 1초로 고정, hello를 1초 ÷ 5 = 200ms 간격으로 전송
 ip ospf dead-interval minimal hello-multiplier 5
```

> ⚠️ 양쪽 인터페이스에 동일하게 적용해야 neighbor 관계 유지됨

### TTL Security (보안 강화)

OSPF 패킷에 TTL=255를 설정하고, 수신 TTL이 임계값 미만이면 폐기 → 원격 스푸핑 공격 방어

```
interface GigabitEthernet 0/1
 ! hops 1 = 동일 세그먼트에서만 수신 허용 (TTL < 254 이면 폐기)
 ip ospf ttl-security hops 1
```

### Prefix Suppression (광고 최적화)

Transit 링크의 IP 주소를 LSA에 광고하지 않아 라우팅 테이블 크기 감소 및 보안 향상

```
! 프로세스 전체 적용
router ospf 1
 prefix-suppression              ! 모든 인터페이스의 Transit 링크 주소 광고 억제

! 인터페이스별 개별 적용
interface GigabitEthernet 0/1
 ip ospf prefix-suppression
```

### MD5 인증 (Authentication)

OSPF 패킷의 무결성 검증 — 동일 Area/이웃 간 동일한 키 필요

```
! 방법 1: 인터페이스별 적용 (세밀한 제어)
interface GigabitEthernet 0/1
 ip ospf message-digest-key 1 md5 CISCO_PASS   ! key-id 1, 패스워드 CISCO_PASS
 ip ospf authentication message-digest

! 방법 2: Area 전체 일괄 적용
router ospf 1
 area 0 authentication message-digest
interface GigabitEthernet 0/1
 ip ospf message-digest-key 1 md5 CISCO_PASS
```

> 출처: [OSPFv2 Cryptographic Authentication Guide](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-routing/b-ip-routing/m_iro-ospfv2-crypto-authen.html)

---

## 정보 확인 (검증 명령어)

```
! 인접 라우터 상태 — State가 FULL이어야 정상, 2WAY는 DROther 간 정상
R# show ip ospf neighbor

! LSDB 전체 목록 — Router/Network/Summary/External LSA 종류 및 갯수 확인
R# show ip ospf database

! OSPF로 학습한 경로만 필터링 — [110/X] 형식으로 Cost 확인 가능
R# show ip route ospf

! 인터페이스별 OSPF 상세 — Hello/Dead 타이머, DR/BDR 역할, Cost, MTU 확인
R# show ip ospf interface GigabitEthernet 0/1

! OSPF 프로세스 전반 정보 — Router-ID, SPF 횟수, Area 멤버십 확인
R# show ip ospf

! 라우팅 프로토콜 전체 요약 — 광고 네트워크, 타이머, 재분배 여부 확인
R# show ip protocol
```

| 명령어 | 주로 확인하는 것 |
|--------|----------------|
| `show ip ospf neighbor` | Full/2Way 상태, Dead Time 소진 여부 |
| `show ip ospf database` | LSA 종류 및 Seq. 번호 — 동기화 여부 |
| `show ip ospf interface` | DR/BDR 역할, Hello/Dead 타이머, Cost |
| `show ip route ospf` | 실제 라우팅 테이블 반영 여부 확인 |
| `show ip ospf` | SPF 재계산 횟수 — 불안정 여부 파악 |

---

## IOS XE 16.x / 17.x 최신 트렌드

### OSPFv3 (IPv6) 설정 방식 변경 (`IOS XE 16.x+`)

IOS 15.x까지는 OSPFv3를 인터페이스에 직접 `ipv6 ospf X area Y`로 적용.  
IOS XE 16.x부터 **Address Family 방식**으로 통합 — IPv4/IPv6를 단일 프로세스로 관리 가능.

```
! IOS 15.x 방식 (Phase 2 자료)
ipv6 router ospf 1
 router-id 1.1.1.1
interface GigabitEthernet0/1
 ipv6 ospf 1 area 0

! IOS XE 16.x Address Family 방식 (권장)
router ospfv3 1
 address-family ipv4 unicast
  router-id 1.1.1.1
 exit-address-family
 address-family ipv6 unicast
  router-id 1.1.1.1
 exit-address-family

interface GigabitEthernet0/1
 ospfv3 1 ipv4 area 0
 ospfv3 1 ipv6 area 0
```

### BFD 연동 (`IOS XE 16.x+`)

OSPF Dead Timer 없이 sub-second 링크 장애 감지

```
! BFD 활성화
interface GigabitEthernet0/1
 bfd interval 300 min_rx 300 multiplier 3

! OSPF 프로세스에 BFD 연동
router ospf 1
 bfd all-interfaces          ! 전체 인터페이스 적용
! 또는 인터페이스별
interface GigabitEthernet0/1
 ip ospf bfd
```

### SHA 인증 (`IOS XE 17.x+`)

IOS 15.x Phase 2 자료는 MD5만 지원. IOS XE 17.x부터 SHA-256/SHA-512 지원.

```
! Key Chain으로 SHA 인증 (IOS XE 17.x)
key chain OSPF_KEY
 key 1
  key-string OSPF_SECRET
  cryptographic-algorithm hmac-sha-256

interface GigabitEthernet0/1
 ip ospf authentication key-chain OSPF_KEY
```

### OSPF SR-TE (Segment Routing) (`IOS XE 17.x+`)

MPLS 없이 소스 기반 경로 제어 — 데이터센터/WAN 최적화 용도

```
! Segment Routing 활성화 (IOS XE 17.x)
router ospf 1
 segment-routing mpls        ! SR-MPLS 활성화
 segment-routing forwarding mpls

! Node SID (라우터별 고유 식별자) 설정
interface Loopback0
 ip address 1.1.1.1 255.255.255.255
 ip ospf 1 area 0
 segment-routing mpls        ! Loopback에 SID 광고
```

> SR-TE는 IOS XE 17.x + 고급 라이선스(DNA Advantage) 필요. Phase 2 자료 범위 외.

---

## OSPF vs EIGRP 비교

| 항목 | OSPF | EIGRP |
|------|------|-------|
| 타입 | Link State | Advanced Distance Vector |
| 표준 | IEEE 표준 | CISCO 전용 |
| AD | 110 | 90 (Internal) |
| Metric | Cost | BW + Delay |
| 알고리즘 | Dijkstra | DUAL |
| Protocol 번호 | 89 | 88 |
