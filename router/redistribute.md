# Redistribute (재분배)

서로 다른 라우팅 프로토콜 간에 라우팅 정보를 교환하는 기능

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
