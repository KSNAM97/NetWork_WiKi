# RIP (Routing Information Protocol)

- Distance Vector 라우팅 프로토콜
- Protocol 번호: **UDP 520**
- AD (Administrative Distance): **120**
- Metric: **Hop-count** (최대 15, 16 = 도달 불가)

---

## RIPv1 vs RIPv2

| 항목 | RIPv1 | RIPv2 |
|------|-------|-------|
| 클래스 | Classful | Classless |
| Subnet Mask | 업데이트에 미포함 | 포함 |
| 전송 방식 | Broadcast `255.255.255.255` | Multicast `224.0.0.9` |
| VLSM/CIDR | 미지원 | 지원 |
| 인증 | 미지원 | Text / MD5 지원 |
| auto-summary | 기본 활성 | 기본 활성 (비활성 권장) |

> ⚠️ RIPv1은 Classful이라 서브넷 마스크를 업데이트에 포함하지 않음 → VLSM 환경에서 사용 불가

---

## 사전 설정 (Pre-config) — 실습 토폴로지

아래 RIPv1/RIPv2 설정 예시는 모두 R1-R2-R3 3대 라우터로 구성된 아래 토폴로지를 기준으로 한다.

```
                          13.13.12.0/24            13.13.23.0/24
                 R1---------------------R2---------------------R3
                  |                                |                                |
         13.13.10.0/24                  13.13.20.0/24                 13.13.30.0/24
```

- R1, R2, R3 각각 Loopback 172 인터페이스에 172.16.x.x/24 대역 부여 (예: R1 = 172.16.1.1, R2 = 172.16.2.2, R3 = 172.16.3.3)
- R1 FastEthernet 0/0 = 13.13.10.254, R2 FastEthernet 0/0 = 13.13.20.254, R3 FastEthernet 0/0 = 13.13.30.254
- R1-R2 구간(Serial 1/0-1/1) = 13.13.12.0/24, R2-R3 구간(Serial 1/0-1/1) = 13.13.23.0/24
- Serial 인터페이스는 encapsulation hdlc, DCE 쪽(R2)에 clock rate 64000 지정

```
! R1 기본 인터페이스 설정 예시
hostname R1
!
interface loopback 172
 ip address 172.16.1.1 255.255.255.0
!
interface fastethernet 0/0
 no shutdown
 ip address 13.13.10.254 255.255.255.0
!
interface serial 1/0
 no shutdown
 encapsulation hdlc
 bandwidth 64
 ip address 13.13.12.1 255.255.255.0
!
```

---

## RIPv1 설정

```
! RIP Process 시작 — 클래스 경계 기반 네트워크 광고
router rip
 version 1
 network 13.0.0.0    ! 13.x.x.x 범위 인터페이스 광고 (클래스 A 전체)
```

### RIPv1 설정 예시

```
# R1
router rip
 version 1
 network 13.0.0.0
!

# R2
router rip
 version 1
 network 13.0.0.0
!

# R3
router rip
 version 1
 network 13.0.0.0
!
```

### debug ip rip로 본 RIPv1 업데이트 (Classful)

```
R1# debug ip rip
RIP protocol debugging is on

RIP: sending  v1 update to 255.255.255.255 via FastEthernet0/0 (13.13.1.254)
RIP: build update entries
      network 172.16.0.0 metric 1
      network 13.13.2.0 metric 2
      network 13.13.3.0 metric 3
      network 13.13.12.0 metric 1
      network 13.13.23.0 metric 2

RIP: sending  v1 update to 255.255.255.255 via Serial1/0 (13.13.12.1)
RIP: build update entries
      network 172.16.0.0 metric 1
      network 13.13.1.0 metric 1

RIP: received v1 update from 13.13.12.2 on Serial1/0
      13.13.2.0 in 1 hops
      13.13.3.0 in 2 hops
      13.13.23.0 in 1 hops
```

> RIPv1은 Classful이므로 업데이트 항목에 SubnetMask가 표시되지 않는다 (`network 13.13.2.0 metric 2` 형식, `/24` 표기 없음).

---

## RIPv2 설정

```
! RIPv2 — Classless, Multicast 224.0.0.9 사용
router rip
 version 2                        ! RIPv2 활성화
 no auto-summary                  ! 자동 요약 비활성화 (VLSM 환경 필수)
 network 13.0.0.0                 ! 광고 네트워크
 passive-interface default        ! 모든 인터페이스 업데이트 송신 차단
 no passive-interface serial 1/0  ! 이웃 라우터 연결 포트만 활성화
```

### RIPv2 설정 예시

```
# R1
router rip
 version 2
 no auto-summary
 network 13.0.0.0
 network 172.16.0.0
 passive-interface loopback 172     ! 루프백은 이웃 없으므로 차단
 passive-interface fastethernet 0/0 ! LAN 포트는 이웃 없으므로 차단
!

# R2
router rip
 version 2
 no auto-summary
 network 13.0.0.0
 passive-interface fastethernet 0/0
!

# R3
router rip
 version 2
 no auto-summary
 network 13.0.0.0
 network 172.16.0.0
 passive-interface loopback 172
 passive-interface fastethernet 0/0
!
```

### debug ip rip로 본 auto-summary 유무 차이

```
! auto-summary 활성 상태(RIPv2 기본값)일 때
R1# debug ip rip
*Mar  1 00:24:28.131: RIP: sending v2 update to 224.0.0.9 via Serial1/0 (13.13.12.1)
*Mar  1 00:24:28.135: RIP: build update entries
*Mar  1 00:24:28.135:   13.13.10.0/24 via 0.0.0.0, metric 1, tag 0    <---- 서브넷은 그대로 광고
*Mar  1 00:24:28.139:   172.16.0.0/16 via 0.0.0.0, metric 1, tag 0    <---- 자동요약으로 Class B 경계(172.16.0.0/16)로 뭉쳐서 광고됨

! no auto-summary 적용 후
R1# debug ip rip
*Mar  1 00:28:41.907: RIP: sending v2 update to 224.0.0.9 via Serial1/0 (13.13.12.1)
*Mar  1 00:28:41.911: RIP: build update entries
*Mar  1 00:28:41.911:   13.13.10.0/24 via 0.0.0.0, metric 1, tag 0
*Mar  1 00:28:41.915:   172.16.1.0/24 via 0.0.0.0, metric 1, tag 0    <---- 원래 서브넷 마스크(/24) 그대로 광고
```

> Distance Vector 프로토콜은 기본적으로 `auto-summary`가 동작한다. 이 기능은 네트워크 정보를 원래의 Class 경계로 요약해서 광고하므로, 불연속 서브넷(discontiguous subnet) 환경에서는 `no auto-summary`로 반드시 비활성화해야 한다.

---

## Loop 방지 메커니즘

### Split-Horizon

- Distance Vector 계열의 기본 Loop 방지 알고리즘
- 어떤 Interface로 학습한 경로는 **동일한 Interface로 다시 광고하지 않음**
- 라우팅 루프 가능성을 원천 차단

```
! Split-Horizon 상태 확인
R# show ip interface serial 1/0
!   Split horizon is enabled   ← 기본 활성
```

### 등가 부하 분산 (Equal-Cost Load Balancing)

- 동일 목적지에 대해 **Metric이 같은 경로가 여러 개**이면 분산
- RIP는 Equal-Cost Load Balancing 지원
- 기본 최대 경로 수: **4개** (IOS 12.2: 최대 6개, IOS 12.4: 최대 16개)

```
! 최대 부하 분산 경로 수 설정
router rip
 maximum-paths 6
```

---

## RIP 타이머

| 타이머 | 기본값 | 설명 |
|--------|--------|------|
| **Update** | 30초 | 전체 라우팅 테이블을 주기적으로 전송 (±15% Jitter 적용) |
| **Invalid** | 180초 | 업데이트 미수신 시 해당 경로를 Invalid 처리 |
| **Holddown** | 180초 | Invalid 이후 더 나쁜 경로 수신을 무시 (루프 방지) |
| **Flushed** | 240초 | 라우팅 테이블에서 해당 경로 완전 삭제 |

---

## Route Poison & Poison Reverse (루프 방지)

- **Route Poison**: 경로 장애 발생 시 Metric을 **16(도달 불가)**으로 즉시 광고
- **Poison Reverse**: Route Poison을 받은 라우터가 해당 경로에 Metric 16으로 응답하여 확인

```
# 인터페이스 shutdown → R2가 감지

*Mar 1: RIP: received v2 update from R1    (1) Route Poison
      13.13.10.0/24 via 0.0.0.0 in 16 hops (inaccessible)

*Mar 1: RIP: sending v2 flash update to R3 (2) Route Poison 전파
      13.13.10.0/24 via 0.0.0.0, metric 16

*Mar 1: RIP: sending v2 flash update to R1 (3) Poison Reverse
      13.13.10.0/24 via 0.0.0.0, metric 16
```

---

## RIPv2 인증 (MD5)

RIPv2는 Key Chain을 이용한 MD5 인증을 지원 → 인증된 라우터 간에만 업데이트 교환

```
! 1단계: Key Chain 정의
key chain RIP_AUTH
 key 20110117                  ! Key ID (양쪽 동일해야 함)
  key-string cisco1234         ! 실제 패스워드

! 2단계: 인터페이스에 인증 적용
interface serial 1/0
 ip rip authentication mode md5          ! 인증 방식: md5 (또는 text)
 ip rip authentication key-chain RIP_AUTH
```

### 인증 확인

```
! 인증 불일치 시 debug 출력
R1# debug ip rip
*Mar 1: RIP: ignored v2 packet from 13.13.12.2 (invalid authentication)

! 인증 성공 시
*Mar 1: RIP: received packet with MD5 authentication

R1# show ip protocol
  Interface    Send  Recv  Triggered RIP  Key-chain
  Serial1/0    2     2                    RIP_AUTH
```

**예시 문제**: R1과 R2 사이 구간(Serial 1/0)에서만 신뢰할 수 없는 제3자의 RIP 업데이트 주입을 막기 위해 인증을 걸고 싶고, R2와 R3 사이 구간은 인증 없이 그대로 두려는 조건이라면? RIP 인증은 프로세스 전역이 아니라 **인터페이스 단위**로 붙는다는 점이 핵심이다.

```
! R1, R2 공통 — 동일한 Key ID/String의 Key Chain 정의
key chain RIP_AUTH
 key 1
  key-string cisco1234

! R1 — R2와 마주보는 인터페이스에만 인증 적용
interface serial 1/0
 ip rip authentication mode md5
 ip rip authentication key-chain RIP_AUTH

! R2 — 동일한 인터페이스에만 적용 (R3 방향 인터페이스는 그대로 둠)
interface serial 1/0
 ip rip authentication mode md5
 ip rip authentication key-chain RIP_AUTH
```

확인: R1-R2 구간은 정상적으로 이웃을 유지하지만, Key를 일부러 다르게 설정하면 `debug ip rip`에서 `ignored v2 packet ... (invalid authentication)`이 찍힌다. R2-R3 인터페이스에는 인증 명령을 넣지 않았으므로 그 구간은 평소처럼 인증 없이 업데이트가 오간다 — 인증이 "프로세스 전체"가 아니라 "그 인터페이스"에만 적용되는 것을 확인하는 것이 이 조건의 핵심이다.

---

## RIP Triggered (변화 시에만 업데이트)

- 기본 RIP는 30초마다 전체 테이블을 전송 → WAN 대역폭 낭비
- `ip rip triggered`: 라우팅 변화 발생 시에만 업데이트 전송 (P2P WAN에 적합)

```
interface serial 1/0
 ip rip triggered    ! 변화 감지 시에만 업데이트 전송
```

---

## RIPv2 Unicast 전송

- 기본 RIPv2는 Multicast(224.0.0.9) 전송
- `neighbor` 명령으로 특정 라우터에만 Unicast 전송 가능

```
! passive-interface로 Multicast 차단 후 neighbor로 Unicast 지정
router rip
 passive-interface serial 1/0   ! Multicast 차단
 neighbor 13.13.12.2            ! 해당 라우터에만 Unicast 전송
```

---

## 정보 확인 (검증 명령어)

```
! RIP로 학습한 경로 확인 — [120/hop] 형식
R# show ip route rip

! 라우팅 프로토콜 설정 확인 — 버전, passive-interface, 타이머 확인
R# show ip protocol

! RIP 패킷 송수신 실시간 확인 — v1/v2 구분, 출발지/목적지 주소 확인
R# debug ip rip
```

| 명령어 | 주로 확인하는 것 |
|--------|----------------|
| `show ip route rip` | RIP 경로 존재 여부 및 Hop count |
| `show ip protocol` | 버전, 광고 네트워크, Passive 설정 |
| `debug ip rip` | 실제 송수신 패킷 내용 (Multicast/Unicast 여부) |
| `show key chain` | MD5 인증 Key 설정 확인 |

---

## IOS XE 16.x / 17.x 관점에서의 RIP

> ⚠️ **RIP는 Legacy 프로토콜** — IOS XE 17.15(2026 현재 최신)까지 지원은 유지되나 신규 설계에는 사용하지 않음

### RIP의 현재 위치

| 항목 | 내용 |
|------|------|
| 마지막 표준 | RIPv2 (RFC 2453, 1998) / RIPng for IPv6 (RFC 2080) |
| IOS XE 지원 | 17.x까지 포함 — 삭제 예고 없음, 단 신규 기능 추가 없음 |
| 실무 대체 | **OSPF** (멀티에리어, 빠른 수렴) 또는 **EIGRP** (CISCO 환경) |
| 유일한 사용처 | 매우 소규모 환경, 레거시 장비 연동, 시험/학습 목적 |

### IOS XE 16.x에서의 RIP 명령어 변화 (없음)

RIP는 IOS 15.x → IOS XE 16.x/17.x로 전환되어도 명령어 문법 변화 없음.  
단, IOS XE 환경에서는 아래와 같이 IPv6 RIPng도 동일 방식으로 설정 가능:

```
! RIPng (IPv6 — IOS XE 포함 동일 문법)
ipv6 unicast-routing
ipv6 router rip RIP_PROCESS

interface GigabitEthernet0/1
 ipv6 rip RIP_PROCESS enable
```

> 📌 **결론**: IOS XE 16.x/17.x 기반 신규 네트워크에서는 **OSPF 또는 EIGRP Named Mode** 사용 권장. RIP는 기존 Phase 2 자료의 문법 그대로 유효하나 확장성·수렴속도 한계로 실무에서 도태됨.
