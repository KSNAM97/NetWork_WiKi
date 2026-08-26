# 경로 필터링 (Route Filtering)

라우팅 업데이트에서 특정 네트워크를 허용하거나 차단하여 라우팅 테이블을 제어한다.

---

## 1. Distribute-list

ACL을 기반으로 라우팅 업데이트를 필터링

```
Router(config)# access-list [번호] permit/deny [네트워크] [Wildcard]
Router(config)# router eigrp/ospf/rip [AS/Process]
Router(config-router)# distribute-list [ACL번호] in/out [Interface]
```

### 예시

```
R1(config)# access-list 10 deny 10.10.10.0 0.0.0.255
R1(config)# access-list 10 permit any
!
R1(config)# router eigrp 100
R1(config-router)# distribute-list 10 in serial 1/0
```

- **in**: 수신하는 라우팅 업데이트에 필터 적용
- **out**: 송신하는 라우팅 업데이트에 필터 적용

---

## 2. Prefix-list

CIDR 기반으로 정밀하게 네트워크 범위를 지정하여 필터링

```
Router(config)# ip prefix-list [이름] seq [번호] permit/deny [prefix/len] [le/ge 옵션]
Router(config)# router eigrp/ospf/rip [AS/Process]
Router(config-router)# distribute-list prefix [이름] in/out [Interface]
```

### 옵션

| 옵션 | 의미 |
|------|------|
| `ge [길이]` | prefix 길이가 지정값 **이상**인 네트워크 매칭 |
| `le [길이]` | prefix 길이가 지정값 **이하**인 네트워크 매칭 |

### 예시

```
# 192.168.0.0/16 범위에서 /24 ~ /32인 네트워크만 허용
R1(config)# ip prefix-list FILTER seq 10 permit 192.168.0.0/16 ge 24 le 32
R1(config)# ip prefix-list FILTER seq 20 deny 0.0.0.0/0 le 32

R1(config)# router ospf 1
R1(config-router)# distribute-list prefix FILTER in
```

---

## 3. Route-map

조건(Match)에 따라 특정 동작(Set)을 실행하는 정책 기반 필터링
- 가장 유연하고 강력한 필터링 도구
- Redistribute, BGP Policy, PBR 등에서도 사용

```
Router(config)# route-map [이름] permit/deny [sequence]
Router(config-route-map)# match [조건]
Router(config-route-map)# set [동작]
```

### 예시: Redistribute 시 특정 네트워크만 배포

```
R1(config)# access-list 10 permit 10.10.10.0 0.0.0.255
!
R1(config)# route-map REDIST permit 10
R1(config-route-map)# match ip address 10
R1(config-route-map)# set metric 100
!
R1(config)# route-map REDIST deny 20
!
R1(config)# router ospf 1
R1(config-router)# redistribute eigrp 100 subnets route-map REDIST
```

### Route-map OR / AND 조건 논리

- **같은 sequence 번호**: match 조건들이 **AND** (모두 일치해야 permit)
- **다른 sequence 번호**: 각 sequence는 **OR** (하나라도 일치하면 permit)

```
# 같은 sequence — AND 조건: ACL 10 AND tag 100이 모두 일치해야 permit
route-map FILTER permit 10
 match ip address 10
 match tag 100

# 다른 sequence — OR 조건: ACL 10 OR ACL 20 중 하나라도 일치하면 permit
route-map FILTER permit 10
 match ip address 10
route-map FILTER permit 20
 match ip address 20
```

### Route-map Tagging (태그 설정)

재분배 루프 방지 및 경로 식별을 위해 사용

```
# EIGRP → OSPF 재분배 시 tag 100 부착
route-map EIGRP_TO_OSPF permit 10
 match ip address prefix-list EIGRP_ROUTES
 set tag 100

router ospf 1
 redistribute eigrp 100 subnets route-map EIGRP_TO_OSPF

# OSPF → EIGRP 재분배 시 tag 100인 경로(OSPF에서 온 경로) 차단 — 루프 방지
route-map OSPF_TO_EIGRP deny 5
 match tag 100

route-map OSPF_TO_EIGRP permit 10
 set tag 200

router eigrp 100
 redistribute ospf 1 metric 10000 100 255 1 1500 route-map OSPF_TO_EIGRP
```

---

## 필터링 도구 비교

| 도구 | 기반 | 유연성 | 주요 사용 |
|------|------|--------|-----------|
| Distribute-list | ACL | 낮음 | 단순 네트워크 필터 |
| Prefix-list | CIDR Prefix | 중간 | 정밀한 prefix 범위 필터 |
| Route-map | Match/Set | 높음 | 조건부 정책, Redistribute |

---

## 정보 확인

```
R# show ip prefix-list
R# show route-map
R# show ip protocols
```

---

## IOS XE 16.x / 17.x 최신 트렌드

### Route-map 조건 확장 (`IOS XE 16.x+`)

IOS 15.x Phase 2 자료의 Route-map은 `match ip address` / `set metric` 위주.  
IOS XE 16.x 이상에서는 **BGP Community, Tag, Extcommunity** 등 다양한 match/set 조건 지원.

```
! IOS XE 16.x — 커뮤니티 기반 필터링 (BGP 연동 시)
route-map FILTER_COMMUNITY permit 10
 match community 100             ! BGP Community 100 매칭
 set local-preference 200

! Tag 기반 필터링 (Redistribute 제어)
route-map TAG_FILTER permit 10
 match tag 100                   ! 태그 100인 경로만 허용
 set metric 1000

route-map TAG_FILTER deny 20     ! 나머지 차단
```

### Prefix-list 와일드카드 패턴 (`IOS XE 16.x+`)

```
! IOS 15.x 기존 방식
ip prefix-list FILTER seq 10 permit 10.0.0.0/8 le 24

! IOS XE 16.x — ge/le 조합으로 정밀 범위 제어
ip prefix-list EXACT_24 seq 10 permit 0.0.0.0/0 ge 24 le 24   ! /24만 허용
ip prefix-list SUPERNET seq 10 permit 0.0.0.0/0 le 8           ! /8 이하 슈퍼넷만
ip prefix-list HOST seq 10 permit 0.0.0.0/0 ge 32              ! 호스트 루트만

! 확인
R# show ip prefix-list detail     ! 각 항목별 hit count 포함
R# show route-map                 ! match/set 결과 카운터 확인
```

### Distribute-list with Route-map (`IOS XE 16.x+`)

IOS 15.x Phase 2 자료에서는 Distribute-list에 ACL만 사용 가능.  
IOS XE 16.x 이상에서는 **Route-map 직접 연결** 지원.

```
! IOS XE 16.x — Distribute-list에 Route-map 연결
router ospf 1
 distribute-list route-map OSPF_FILTER in   ! 수신 경로 필터링

router eigrp CAMPUS
 address-family ipv4 unicast autonomous-system 100
  topology base
   distribute-list route-map EIGRP_FILTER in
```
