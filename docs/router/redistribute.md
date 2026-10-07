# Redistribute (재분배)

서로 다른 라우팅 프로토콜 간에 라우팅 정보를 교환하는 기능

---

## 사전 설정 (Pre-config)

Redistribute 실습을 위한 기본 토폴로지 — R1~R5 5대의 라우터로 구성되며, R1-R2-R3는 Frame-Relay 멀티포인트(13.13.9.0/24)로 연결되고 R1-R3는 별도 P2P 서브인터페이스(13.13.10.0/24)로도 연결된다. **R4는 RIPv2**, **R5는 EIGRP 100**을 각각 독자적으로 운영하며, 이 둘을 재분배로 연결하는 것이 핵심 실습이다.

- R1: Lo0 13.13.1.1/24, Fa0/0 150.1.13.1/24(R4 방향), Fa0/1 13.13.11.1/24, S1/0.123(F-R Multipoint) 13.13.9.1/24, S1/1(F-R) 13.13.10.1/24(R3 방향)
- R2: Lo0 13.13.2.2/24, Fa0/1 13.13.12.2/24, S1/0.123(F-R Multipoint) 13.13.9.2/24
- R3: Lo0 13.13.3.3/24, Fa0/0 150.3.13.3/24(R5 방향), Fa0/1 13.13.13.3/24, S1/0.123(F-R Multipoint) 13.13.9.3/24, S1/1(F-R) 13.13.10.3/24(R1 방향)
- **R4 (RIP 도메인)**: Lo0 13.13.4.4/24, Lo150 150.100.1.254/24, Lo10(10.1.0-3.4/24), Lo172(172.16.4-7.4/24), Lo192(192.168.8-11.4/24), Lo199(199.171.1-16.4/24 다중 secondary), Fa0/0 150.1.13.4/24(R1 방향), Fa0/1 13.13.14.4/24 — `router rip` / `version 2` / 각 대역 `network` / `no auto-summary`
- **R5 (EIGRP 100 도메인)**: Lo0 13.13.5.5/24, Lo4 4.1.1.5/24, Lo128(128.28.2.5, 128.128.1.5), Lo198(198.198.x.5, 198.2.x.5, 198.1.1.5/30 다중 secondary), Fa0/0 150.3.13.5/24(R3 방향), Fa0/1 13.13.15.5/24 — `router eigrp 100` / `passive-interface default` + `no passive-interface fastethernet 0/0` / 각 대역 `network` / `no auto-summary`

공통 초기화(`no ip domain-lookup`, `enable secret cisco`, `line con/vty password cisco`)는 모든 라우터에 적용된다.

```
! R4 — RIPv2 도메인 (재분배 대상 원본)
router rip
 version 2
 network 10.0.0.0
 network 13.0.0.0
 network 150.1.0.0
 network 172.16.0.0
 network 150.100.0.0
 network 192.168.8.0
 network 192.168.9.0
 network 192.168.10.0
 network 192.168.11.0
 network 199.171.1.0
 network 199.171.2.0
 network 199.171.3.0
 network 199.171.4.0
 network 199.171.5.0
 network 199.171.6.0
 network 199.171.7.0
 network 199.171.8.0
 network 199.171.9.0
 network 199.171.10.0
 network 199.171.11.0
 network 199.171.12.0
 network 199.171.13.0
 network 199.171.14.0
 network 199.171.15.0
 network 199.171.16.0
 no auto-summary

! R5 — EIGRP 100 도메인 (재분배 대상 원본)
router eigrp 100
 no auto-summary
 passive-interface default
 no passive-interface fastethernet 0/0
 network 13.13.5.5 0.0.0.0
 network 13.13.15.5 0.0.0.0
 network 4.1.1.5 0.0.0.0
 network 128.28.2.5 0.0.0.0
 network 128.128.1.5 0.0.0.0
 network 150.1.3.5 0.0.0.0
 network 150.3.13.5 0.0.0.0
 network 198.1.1.5 0.0.0.0
 network 198.2.0.0 0.0.255.255
 network 198.198.0.0 0.0.255.255
```

---

## 개념

- 서로 다른 라우팅 프로토콜이 공존하는 환경에서 필요
- Redistribute를 설정한 라우터가 **경계 라우터(ASBR)** 역할

```
EIGRP 도메인 ──────────── ASBR ──────────── OSPF 도메인
                     (Redistribute)
```

---

## 기본 설정

### EIGRP → OSPF 재분배

```
R(config)# router ospf 1
R(config-router)# redistribute eigrp 100 subnets metric 100
! metric-type 기본값은 E2 (고정 Metric)
! metric-type 1 = 누적 Metric (경로가 복수인 경우 정확한 비교에 유리)
R(config-router)# redistribute eigrp 100 subnets metric-type 1
```

### OSPF → EIGRP 재분배

```
R(config)# router eigrp 100
R(config-router)# redistribute ospf 1 metric 10000 100 255 1 1500
```

> EIGRP 재분배 시 metric 값 지정 필수: `[bandwidth] [delay] [reliability] [load] [MTU]`

### OSPF ↔ RIPv2 재분배

```
# OSPF → RIPv2
R(config)# router rip
R(config-router)# redistribute ospf 1 metric 5    ! RIP metric (홉 수) 직접 지정

# RIPv2 → OSPF
R(config)# router ospf 1
R(config-router)# redistribute rip metric-type 1 subnets
! 또는 metric 직접 지정
R(config-router)# redistribute rip subnets metric 100
```

### Connected 네트워크 재분배

```
R(config)# router ospf 1
R(config-router)# redistribute connected subnets

# Route-map으로 특정 인터페이스만 재분배
R(config)# route-map CONNECTED_FILTER permit 10
R(config-route-map)# match interface fastethernet 0/0    ! 특정 인터페이스만 허용

R(config)# router ospf 1
R(config-router)# redistribute connected subnets route-map CONNECTED_FILTER
```

### Static 경로 재분배

```
R(config)# router ospf 1
R(config-router)# redistribute static subnets
```

---

## Route-map을 활용한 필터링 재분배

특정 네트워크만 선택적으로 재분배

```
R(config)# access-list 10 permit 192.168.10.0 0.0.0.255
R(config)# access-list 10 permit 192.168.20.0 0.0.0.255
!
R(config)# route-map EIGRP_TO_OSPF permit 10
R(config-route-map)# match ip address 10
R(config-route-map)# set metric 50
!
R(config)# route-map EIGRP_TO_OSPF deny 20
!
R(config)# router ospf 1
R(config-router)# redistribute eigrp 100 subnets route-map EIGRP_TO_OSPF
```

**예시 문제**: R1이 RIPv2 네트워크 정보를 OSPF 환경으로 재분배해야 하는데, RIPv2 쪽에는 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 같은 사설(Private) 네트워크 대역도 섞여 있다. 이 사설 네트워크 정보는 OSPF 환경으로 재분배되지 않도록 막고, 나머지 네트워크는 그대로 재분배하려면?

```
! 차단할 사설 네트워크 범위를 Prefix-list로 지정
ip prefix-list PRIVATE permit 10.0.0.0/8 le 32
ip prefix-list PRIVATE permit 172.16.0.0/12 le 32
ip prefix-list PRIVATE permit 192.168.0.0/16 le 32
!
route-map RIPv2-->OSPF_REDI deny 10
 match ip address prefix-list PRIVATE      ! PRIVATE에 매칭되는 네트워크는 차단
!
route-map RIPv2-->OSPF_REDI permit 20      ! 나머지 모든 네트워크는 재분배 허용
!
router ospf 1
 redistribute rip subnets route-map RIPv2-->OSPF_REDI
```

확인: `R2# show ip route ospf | include 10.1.` / `172` / `192` 로 조회 시 사설 네트워크 정보는 나타나지 않아야 한다.

---

## Metric 설정

### Default Metric 설정

```
R(config)# router ospf 1
R(config-router)# default-metric 100
```

### 재분배 시 직접 Metric 지정

```
R(config-router)# redistribute eigrp 100 subnets metric 200
```

---

## 양방향 재분배 시 루프 주의

양방향 재분배 시 **라우팅 루프** 발생 가능

**해결 방법**
- Route-map + Tag를 사용하여 이미 재분배된 경로 식별 후 차단

```
R(config)# route-map OSPF_TO_EIGRP deny 10
R(config-route-map)# match tag 100          ← EIGRP에서 온 경로 차단

R(config)# route-map OSPF_TO_EIGRP permit 20
R(config-route-map)# set tag 200

R(config)# route-map EIGRP_TO_OSPF deny 10
R(config-route-map)# match tag 200          ← OSPF에서 온 경로 차단

R(config)# route-map EIGRP_TO_OSPF permit 20
R(config-route-map)# set tag 100
```

**예시 문제**: R3이 OSPF에 포함된 네트워크 정보를 EIGRP 환경으로 재분배하는데, "199.171.0.0/24 ~ 199.171.7.0/24" 대역에는 Tag 100을 표시해야 하고, 그 외 "199.171.x.0/24" 나머지 대역은 아예 재분배에서 제외해야 하며, 199.171.x.0/24를 제외한 모든 나머지 네트워크는 정상적으로 재분배되어야 한다면?

```
ip prefix-list NET199_TAG permit 199.171.0.0/21 le 32   ! Tag 부착 대상 범위 (.0~.7)
ip prefix-list NET199_DENY permit 199.171.0.0/16 le 32  ! 199.171.x.0/24 전체 범위
!
route-map OSPF-->EIGRP_REDI permit 10
 match ip address prefix-list NET199_TAG
 set tag 100                        ! .0~.7 대역만 Tag 100 부착
!
route-map OSPF-->EIGRP_REDI deny 20
 match ip address prefix-list NET199_DENY   ! Tag 안 붙은 나머지 199.171.x.0/24는 차단
!
route-map OSPF-->EIGRP_REDI permit 30       ! 199.171.x.0/24 외 나머지는 모두 허용
!
router eigrp 100
 redistribute ospf 1 metric 100000 10 255 1 1500 route-map OSPF-->EIGRP_REDI
```

확인: `R5# show ip eigrp topology | include tag is 100` 으로 .0~.7 대역에만 Tag 100이 붙었는지, `show ip route eigrp | include 199` 로 나머지 199.171.x.0/24 대역이 라우팅 테이블에서 빠졌는지 확인한다. route-map의 permit/deny 순서(구체적 매칭 → 차단 → 허용)가 핵심이다.

---

## 정보 확인

```
R# show ip route
R# show ip route ospf
R# show ip route eigrp
R# show ip protocols
```

---

## IOS XE 16.x / 17.x 최신 트렌드

### Prefix-list 기반 재분배 필터링 (`IOS 15.x / Phase 2 자료`)

재분배 시 특정 네트워크만 광고하거나 차단할 때 route-map + prefix-list 조합 사용

```
! 허용할 네트워크만 Prefix-list로 정의
ip prefix-list FA0/1 permit 13.13.12.0/24

! route-map에서 prefix-list 매칭
route-map OSPF_CONN_REDI permit 10
 match ip address prefix FA0/1
!

! 재분배 시 route-map 적용
router ospf 1
 redistribute connected subnets route-map OSPF_CONN_REDI
```

특정 인터페이스 기준 필터링:

```
! FastEthernet 0/1에 연결된 네트워크만 재분배
route-map OSPF_CONN_REDI permit 10
 match interface fastethernet 0/1
!
router ospf 1
 redistribute connected subnets route-map OSPF_CONN_REDI
```

차단(deny) + 나머지 허용 패턴:

```
! Loopback 네트워크는 재분배 차단, 나머지는 허용
route-map OSPF_CONN_REDI deny 10
 match interface loopback100 loopback200
!
route-map OSPF_CONN_REDI permit 20
!
router ospf 1
 redistribute connected subnets route-map OSPF_CONN_REDI
```

확인:

```
R# show ip route ospf
R# show route-map
```

---

### OSPF → EIGRP Named Mode 재분배 (`IOS XE 16.x+`)

Phase 2 자료의 Classic Mode 재분배와 문법이 다름.

```
! IOS 15.x Classic Mode (Phase 2 자료)
router eigrp 100
 redistribute ospf 1 metric 10000 100 255 1 1500

! IOS XE 16.x Named Mode — topology base 내부에서 설정
router eigrp CAMPUS
 address-family ipv4 unicast autonomous-system 100
  topology base
   redistribute ospf 1 metric 10000 100 255 1 1500
   redistribute static metric 10000 100 255 1 1500
```

### 태그(Tag) 기반 재분배 루프 방지 (`IOS XE 16.x+`)

다중 재분배 포인트에서 루프 방지를 위한 표준 패턴

```
! ASBR1 — OSPF → EIGRP 재분배 시 Tag 부착
route-map OSPF_TO_EIGRP permit 10
 match ip address prefix-list OSPF_ROUTES
 set tag 200                    ! OSPF에서 온 경로임을 Tag로 표시

router eigrp CAMPUS
 address-family ipv4 unicast autonomous-system 100
  topology base
   redistribute ospf 1 route-map OSPF_TO_EIGRP

! ASBR2 — EIGRP → OSPF 재분배 시 Tag 200 차단 (루프 방지)
route-map EIGRP_TO_OSPF deny 5
 match tag 200                  ! OSPF에서 온 경로는 다시 OSPF로 보내지 않음

route-map EIGRP_TO_OSPF permit 10
 set tag 100

router ospf 1
 redistribute eigrp 100 subnets route-map EIGRP_TO_OSPF
```

### BGP 연동 재분배 (`IOS XE 16.x+ — 엔터프라이즈 환경`)

```
! 내부 IGP 경로를 BGP로 재분배 (IOS XE 16.x+)
router bgp 65001
 address-family ipv4 unicast
  redistribute ospf 1 match internal external 1 external 2
  redistribute eigrp 100

! BGP 경로를 OSPF로 재분배
router ospf 1
 redistribute bgp 65001 subnets route-map BGP_TO_OSPF
```

```
! 확인 (IOS XE 16.x+ 추가 명령어)
R# show ip bgp                          ! BGP 테이블
R# show ip ospf database external       ! 재분배된 외부 경로 LSA 확인
R# show ip eigrp topology | include Ex  ! External EIGRP 경로 확인
```
