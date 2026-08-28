# EIGRP (Enhanced Interior Gateway Routing Protocol)

- CISCO 전용 고급 Distance Vector 라우팅 프로토콜
- Protocol 번호: **88**
- AD (Administrative Distance): Internal **90**, External **170**
- Metric: Bandwidth + Delay (기본값)

---

## 기본 설정

```
Router(config)# router eigrp [AS번호]             ! AS번호는 동일 도메인 내 라우터 전체 동일해야 함
Router(config-router)# no auto-summary            ! 클래스풀 자동 요약 비활성화 (VLSM 환경 필수)
Router(config-router)# passive-interface default  ! 모든 인터페이스에서 Hello 송신 차단
Router(config-router)# no passive-interface [if]  ! 이웃을 맺을 인터페이스만 개별 활성화
Router(config-router)# network [IP] [Wildcard]    ! 광고할 네트워크 (0.0.0.0 wildcard = 호스트 정확 매칭)
```

### 설정 예시

```
R1(config)# router eigrp 100
R1(config-router)# no auto-summary               ! VLSM 지원을 위해 반드시 비활성화
R1(config-router)# passive-interface default      ! 기본적으로 모든 포트 비활성 → 불필요한 Hello 차단
R1(config-router)# no passive-interface serial 1/0       ! WAN 연결 포트만 이웃 허용
R1(config-router)# no passive-interface fastethernet 0/0 ! LAN측 다른 라우터 연결 포트 허용
R1(config-router)# network 13.13.1.1 0.0.0.0    ! 호스트 IP 정확 지정 (해당 인터페이스만 광고)
R1(config-router)# network 150.1.13.1 0.0.0.0
```

> `passive-interface default` → 모든 인터페이스에서 EIGRP 업데이트 비활성화  
> `no passive-interface [if]` → 해당 인터페이스만 업데이트 활성화

---

## 부하 분산 (Load Balancing)

### 등가 부하 분산 (Equal-Cost Load Balancing)

- 동일 목적지에 FD(Feasible Distance)가 같은 경로가 여러 개이면 분산
- 기본 최대 경로: **4개** (IOS 12.2: 최대 6개, IOS 12.4: 최대 16개)

```
router eigrp 100
 maximum-paths 8   ! 최대 분산 경로 수 지정
```

### 비등가 부하 분산 (Unequal-Cost Load Balancing) — EIGRP만 지원

- `variance` 명령어로 Successor FD의 배수 이내의 Feasible Successor 경로도 분산에 포함
- variance 2 = Successor FD × 2 이하의 FS 경로를 추가 사용

```
router eigrp 100
 variance 2   ! FD의 2배 이내 경로를 분산에 포함
```

> ⚠️ FS 조건(AD < Successor FD)을 만족해야만 Unequal-Cost 분산 대상이 됨

**예시 문제**: 어떤 목적지 네트워크로 R1을 통한 Successor 경로(FD 2,195,456)와 R2를 통한 경로(AD 2,349,056)가 동시에 존재한다. R2 경로를 Feasible Successor(백업 경로)로 인정받아 부하 분산에도 포함시키려면 어떤 조건과 설정이 필요한가?

- FS 조건 확인: R2 경로의 AD(2,349,056) < Successor FD(2,195,456) → **조건 불만족** (AD가 오히려 더 큼) → 이 경로는 FS가 될 수 없고 라우팅 루프 방지를 위해 Topology Table에서 대기조차 하지 못한다.
- 만약 AD가 FD보다 작아 FS 조건을 만족하는 경우라면, variance로 분산에 포함시킬 수 있다.

```
! FD의 2배 이내 Feasible Successor까지 부하 분산에 포함 (FS 조건 만족 경로에 한함)
router eigrp 100
 variance 2
 maximum-paths 4
```

확인: `R# show ip eigrp topology`에서 Successor는 `P`(Passive) 상태로 표시되고, FS 조건을 만족하지 못하는 경로는 `show ip eigrp topology all-links`에만 나타난다 — 이 경로는 variance를 아무리 크게 줘도 분산 대상에 포함되지 않는다.

---

## IOS XE 최신 트렌드 — Named Mode

> IOS XE 16.x 이상에서 권장하는 방식. Wide Metrics, IPv6 동시 지원, VRF Lite 등 신기능은 Named Mode에서만 사용 가능.
>
> 출처: [Cisco IOS XE 17.x EIGRP Configuration Guide](https://www.cisco.com/c/en/us/support/docs/ip/enhanced-interior-gateway-routing-protocol-eigrp/200156-Configure-EIGRP-Named-Mode.html)

### Classic Mode vs Named Mode

| 항목 | Classic Mode | Named Mode |
|------|-------------|------------|
| 프로세스 생성 | `router eigrp [AS]` | `router eigrp [NAME]` |
| 주소 패밀리 | IPv4 기본, IPv6 별도 | address-family로 통합 |
| Wide Metrics | 미지원 | 지원 (64bit) |
| IPv6 VRF | 미지원 | 지원 |
| 신규 기능 | 추가 없음 | 지속 업데이트 |

### Named Mode 설정

```
! EIGRP Named Mode — IOS XE 16.x+ 권장 방식
router eigrp CORP_NET               ! 인스턴스 이름 (AS번호 아님)
 !
 address-family ipv4 unicast autonomous-system 1   ! IPv4 주소 패밀리 + AS번호 지정
  eigrp router-id 1.1.1.1           ! Router-ID 명시 (루프백 기반 권장)
  network 10.0.0.0 0.255.255.255    ! 광고할 네트워크
  passive-interface default         ! 모든 인터페이스 수신 전용
  no passive-interface Gi0/1        ! EIGRP 이웃을 맺을 인터페이스만 활성화
  !
  af-interface GigabitEthernet0/1
   summary-address 10.10.0.0 255.255.0.0   ! Named Mode 요약 경로
  exit-af-interface
  !
  topology base
   redistribute connected           ! 직접 연결 경로 재분배
  exit-af-topology
  !
  eigrp stub connected summary      ! Stub 라우터 설정
  no shutdown                       ! 프로세스 활성화 (필수)
 exit-address-family
 !
 address-family ipv6 unicast autonomous-system 1   ! IPv6도 동일 인스턴스로 관리
  eigrp router-id 1.1.1.1
  no shutdown
 exit-address-family
```

### Named Mode 확인

```
R# show eigrp address-family ipv4 neighbors
R# show eigrp address-family ipv4 topology
R# show eigrp address-family ipv4 interfaces
```

---

## 정보 확인

```
R# show ip eigrp neighbor      ← 인접 라우터 확인
R# show ip eigrp topology      ← Topology Table 확인
R# show ip route eigrp         ← EIGRP 경로만 확인
R# show ip protocol            ← 라우팅 프로토콜 정보
```

---

## 주소 요약 (Summary)

라우팅 테이블 축소 및 SIA 예방 목적

```
R3(config)# interface fastethernet 0/0
R3(config-if)# ip summary-address eigrp 100 13.13.0.0 255.255.0.0
```

### Default Route 배포

```
R3(config)# interface fastethernet 0/0
R3(config-if)# ip summary-address eigrp 100 0.0.0.0 0.0.0.0
```

---

## Offset-list

특정 네트워크의 Metric 값을 **증가**시켜 경로를 변경

> ⚠️ Offset-list는 Metric 증가만 가능 (감소 불가)

```
R1(config)# access-list 1 permit 13.13.14.0 0.0.0.255
R1(config)# router eigrp 100
R1(config-router)# offset-list 1 in 600000 serial 1/1
```

- **in**: 수신 방향으로 들어오는 업데이트에 적용
- **out**: 송신 방향으로 나가는 업데이트에 적용

**예시 문제**: R5(13.13.15.0/24)가 R4(13.13.14.0/24)로 통신할 때, 현재 최적 경로는 R3 → R1(Serial1/1) 구간을 거치지만, 이를 R3 → R2(Serial1/0.123) 경유로 바꾸고 싶다. EIGRP는 Metric을 직접 지정할 수 없으므로 특정 경로의 Metric을 "증가"시켜 상대적으로 다른 경로가 유리해지도록 유도해야 한다.

```
! R3에서 R1 방향(Serial1/1)으로 들어오는 13.13.14.0/24 경로의 Metric을 인위적으로 증가
R3(config)# access-list 1 permit 13.13.14.0 0.0.0.255
R3(config)# router eigrp 100
R3(config-router)# offset-list 1 in 600000 serial 1/1
```

확인: `R3# show ip route eigrp` 에서 적용 전에는 `13.13.14.0/24 ... via 13.13.10.1, Serial1/1`이 최적 경로였다가, 적용 후에는 `via 13.13.9.2, Serial1/0.123`으로 바뀐다. `R5# traceroute 13.13.14.4 source 13.13.15.5`로 실제 경유 라우터가 R2를 거치는지 확인. offset-list는 Metric을 감소시킬 수 없으므로, "이 경로를 못 쓰게 만들어서" 상대적으로 다른 경로를 유리하게 만드는 방식임에 유의.

---

## Hello / Hold Interval

| 구간 | Hello | Hold |
|------|-------|------|
| LAN (Ethernet/FastEthernet) | 5초 | 15초 |
| WAN (PPP/HDLC/F-R P2P) | 5초 | 15초 |
| WAN (F-R Multipoint/NBMA) | 60초 | 180초 |

```
R(config-if)# ip hello-interval eigrp 100 5
R(config-if)# ip hold-time eigrp 100 15
```

---

## DUAL 알고리즘 (Diffusing Update Algorithm)

### EIGRP 3가지 테이블

| 테이블 | 설명 |
|--------|------|
| **Neighbor Table** | Hello로 발견된 EIGRP 이웃 라우터 목록 |
| **Topology Table** | 이웃에게 수신한 모든 네트워크 경로 및 Metric 저장 |
| **Routing Table** | Topology Table에서 최적 경로(Successor)를 선택해 등록 |

### DUAL 핵심 용어

| 용어 | 설명 |
|------|------|
| **FD (Feasible Distance)** | 자신에서 목적지까지의 최적 Metric (Routing Table에 등록되는 값) |
| **AD / RD (Advertised Distance)** | Next-hop 라우터가 목적지까지 갖는 Metric (이웃이 알려주는 값) |
| **Successor** | 현재 사용 중인 최적 경로 — FD가 가장 작은 경로 |
| **Feasible Successor (FS)** | 백업 경로 — **AD < Successor의 FD** 조건 만족 시 등록 |

### DUAL 경로 선출 과정

1. 이웃에게 수신한 모든 경로를 Topology Table에 저장
2. FD가 가장 작은 경로 → **Successor** (Routing Table 등록)
3. AD < Successor의 FD 조건 만족 경로 → **Feasible Successor** (Topology Table 대기)
4. Successor 장애 시 FS가 즉시 Routing Table에 승격 (Query 없이 빠른 수렴)
5. FS도 없으면 Active 상태로 전환 → Query 전파 시작

```
! Topology Table 확인 — Successor(P)/Active 상태 및 FD, AD 확인
R# show ip eigrp topology
R# show ip eigrp topology all-links   ! FS 조건 미충족 경로도 표시
```

### EIGRP 5가지 PDU

| PDU | 설명 |
|-----|------|
| **Hello** | 이웃 발견 및 유지, Neighbor Table 등록 |
| **Update** | 라우팅 정보 전달 — 변화 발생 시에만 전송 (부분 업데이트) |
| **Query** | Successor 장애 + FS 없음 → 이웃에게 대체 경로 문의 |
| **Reply** | Query에 대한 응답 — 대체 경로 유무 확인 |
| **Ack** | Update/Query/Reply 수신 확인 (Unicast로 전송) |

> Ack 미수신 시 최대 **16회** 재전송 후 Neighbor 관계 해제

---

## SIA (Stuck In Active)

EIGRP 경로 장애 시 Query-Reply 과정에서 Reply를 받지 못해 경로 계산이 완료되지 않는 현상

### EIGRP 경로 상태

| 상태 | 설명 |
|------|------|
| **Passive** | 정상 상태 — 경로가 안정적으로 계산된 상태 |
| **Active** | 장애 상태 — Successor 없어 Query를 전송하며 경로 탐색 중 |

### SIA 발생 과정

1. Successor 경로 장애 감지
2. Feasible Successor 없음 → Active 상태로 전환
3. Neighbor에게 Query 전송
4. **180초(기본)** 내 Reply 미수신 → SIA 판단
5. Reply 미수신 Neighbor와 인접성 해제

### SIA 원인

- 저속 WAN / 네트워크 혼잡
- 높은 패킷 손실
- Router CPU/메모리 과부하
- 과도한 Query 확산 (대규모 네트워크)

### SIA 해결 방법

**1. Active Timer 조정**
```
Router(config-router)# timers active-time 10
```

**2. EIGRP Stub** (Spoke Router에 설정)
```
Router(config-router)# eigrp stub
```
- Stub Router로는 Query를 전송하지 않음

**3. 주소 요약**
- 요약된 네트워크는 상세 경로 정보가 없으므로 Query 대신 즉시 Reply 반환

---

## CIDR & 자동 요약

- **CIDR**: 클래스 기반이 아닌 서브넷 마스크 기반 라우팅
- **no auto-summary**: 자동 요약 비활성화 (권장)
- 자동 요약 시 서로 다른 클래스 간 통신에서 문제 발생 가능

---

## 정보 확인 (검증 명령어)

```
! 이웃 라우터 목록 확인 — Uptime, Queue, SRTT, RTO 포함
R# show ip eigrp neighbors

! 라우팅 테이블의 EIGRP 항목 확인 — Successor / Feasible Distance 확인
R# show ip eigrp topology

! Active 상태의 경로 확인 — SIA 발생 여부 파악
R# show ip eigrp topology active

! 인터페이스별 Hello/Dead 타이머 및 AS번호 확인
R# show ip eigrp interfaces

! 트래픽 카운터 — Hello/Update/Query/Reply 수신·전송 패킷 수
R# show ip eigrp traffic

! Named Mode 사용 시 (IOS XE 16.x+)
R# show eigrp address-family ipv4 neighbors
R# show eigrp address-family ipv4 topology
R# show eigrp address-family ipv4 interfaces
```

| 명령어 | 주로 확인하는 것 |
|--------|----------------|
| `show ip eigrp neighbors` | 이웃 성립 여부, Hold time 소진 여부 |
| `show ip eigrp topology` | Successor / FS 존재 여부, FD·AD 값 |
| `show ip eigrp topology active` | SIA 발생 여부 및 미응답 이웃 파악 |
| `show ip eigrp interfaces` | 인터페이스별 Hello 타이머 적용 확인 |
| `show ip route eigrp` | 실제 라우팅 테이블에 반영된 EIGRP 경로 |

---

## IOS XE 16.x / 17.x 최신 트렌드

### EIGRP Named Mode — Wide Metrics (`IOS XE 16.x+`)

Phase 2 자료까지는 Classic Mode(`router eigrp 100`) 방식. IOS XE 16.x부터 Named Mode가 표준으로 전환됨.

| 항목 | Classic Mode (IOS 15.x) | Named Mode (IOS XE 16.x+) |
|------|------------------------|--------------------------|
| 설정 진입 | `router eigrp 100` | `router eigrp NAME` |
| AS 번호 | 직접 입력 | `address-family` 내부에서 지정 |
| Wide Metrics | 미지원 | **지원** (10Gbps 이상 정확한 Metric 계산) |
| IPv6 VRF Lite | 불가 | 가능 |
| 권장 여부 | IOS 15.x까지 | **IOS XE 16.x 이상 권장** |

```
! IOS XE 16.x Named Mode 설정
router eigrp CAMPUS                    ! AS 이름 (문자열)
 !
 address-family ipv4 unicast autonomous-system 100
  !
  af-interface GigabitEthernet0/1
   hello-interval 5
   hold-time 15
  exit-af-interface
  !
  topology base
   redistribute ospf 1 metric 10000 100 255 1 1500
  exit-af-topology
  !
  eigrp router-id 1.1.1.1             ! Router-ID 명시 권장
  eigrp stub connected summary        ! Stub 설정
 exit-address-family
```

```
! Named Mode 확인 명령어 (IOS XE 16.x+)
R# show eigrp address-family ipv4 neighbors
R# show eigrp address-family ipv4 topology
R# show eigrp address-family ipv4 interfaces detail
R# show eigrp address-family ipv4 accounting    ! Wide Metrics 확인
```

### BFD 연동 (`IOS XE 16.x+`)

EIGRP Hello 타이머보다 빠른 링크 장애 감지 (sub-second)

```
! BFD 활성화 (인터페이스별)
interface GigabitEthernet0/1
 bfd interval 300 min_rx 300 multiplier 3   ! 300ms × 3 = 900ms 이내 감지

! EIGRP에 BFD 연동
router eigrp CAMPUS
 address-family ipv4 unicast autonomous-system 100
  af-interface GigabitEthernet0/1
   bfd                                ! BFD로 이웃 감지
  exit-af-interface
```

### Key Chain 기반 인증 (`IOS XE 17.x — Named Mode 권장`)

```
! Key Chain 정의 (SHA-256 지원 — IOS XE 17.x)
key chain EIGRP_AUTH
 key 1
  key-string EIGRP_SECRET
  cryptographic-algorithm hmac-sha-256   ! IOS 15.x는 MD5만 지원

! Named Mode에서 인증 적용
router eigrp CAMPUS
 address-family ipv4 unicast autonomous-system 100
  af-interface GigabitEthernet0/1
   authentication mode hmac-sha-256 EIGRP_AUTH
  exit-af-interface
```

> IOS 15.x Phase 2 자료에서는 `ip authentication key-chain eigrp 100 CHAIN` (MD5)만 사용 가능

### Classic Mode MD5 인증 (`IOS 15.x / Phase 2 자료`)

```
! 1단계: Key Chain 정의
key chain EIGRP_AUTH
 key 20110117          ! Key ID
  key-string CCNA1234  ! 패스워드

! 2단계: 인터페이스에 MD5 인증 적용
interface serial 1/0
 ip authentication mode eigrp 100 md5
 ip authentication key-chain eigrp 100 EIGRP_AUTH
```

> IOS XE 16.x+ Named Mode에서는 `authentication mode hmac-sha-256` 권장

---

## EIGRP Unicast 전송

기본 EIGRP는 Multicast(224.0.0.10)로 Hello를 전송. `neighbor` 명령으로 특정 라우터에만 Unicast 전송 가능.

```
router eigrp 100
 neighbor 13.13.12.2 serial 1/0   ! 해당 IP로만 Unicast 전송 (인터페이스 필수 지정)
```

> NBMA 환경(Frame-Relay) 또는 Multicast가 지원되지 않는 구간에서 사용
