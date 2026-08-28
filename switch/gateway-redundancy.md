# Gateway 이중화

기본 게이트웨이의 장애 시 자동으로 다른 라우터가 역할을 승계하는 기술

---

## FHRP (First Hop Redundancy Protocol)

| 프로토콜 | 표준 | 가상 MAC | Hello/Dead | 특징 |
|----------|------|---------|-----------|------|
| **HSRP** | CISCO | `0000.0c07.acXX` | 3초/10초 | 가장 많이 사용 |
| **VRRP** | IEEE | `0000.5e00.01XX` | 1초/3초 | 표준, preempt 기본 활성 |
| **GLBP** | CISCO | 여러 가상 MAC | - | 진정한 로드밸런싱 |

---

## HSRP (Hot Standby Router Protocol)

### 동작 원리

```
          가상 IP: 200.20.2.254
          가상 MAC: 0000.0c07.ac01
               │
    ┌──────────┴──────────┐
    │                     │
  G/W1 (Active)        G/W2 (Standby)
  Priority 150         Priority 100
  200.20.2.251         200.20.2.252
```

- **Hello-interval**: 3초 / **Dead-interval**: 10초
- **Multicast**: 224.0.0.2 / **UDP 포트**: 1985
- 우선순위가 높은 라우터가 **Active** 선출

### 기본 설정

```
# G/W1 (Primary)
interface fastethernet 0/1
 standby 1 ip 200.20.2.254      ← 가상 IP
 standby 1 priority 150         ← 높을수록 Active 우선 (기본 100)
 standby 1 timer 1 3            ← Hello 1초, Hold 3초
!

# G/W2 (Backup)
interface fastethernet 0/1
 standby 1 ip 200.20.2.254
 standby 1 priority 100
 standby 1 timer 1 3
```

### Preempt (선점권)

장애 복구 후 우선순위가 높은 라우터가 **자동으로 Active 권한을 되찾는 기능**

```
interface fastethernet 0/1
 standby 1 preempt delay minimum 60    ← 복구 후 60초 뒤 Active 재취득
```

> ⚠️ `preempt`를 설정하지 않으면 복구 후에도 G/W2가 Active를 유지

### Interface Tracking (WAN 구간 장애 감지)

HSRP는 기본적으로 **내부 LAN** 구간 장애만 감지한다.  
`track` 기능으로 WAN 구간 장애 발생 시에도 Gateway 전환 가능

```
# 방법 1: 직접 Interface 추적
interface fastethernet 0/1
 standby 1 track serial 1/0.12 100    ← WAN 장애 시 Priority 100 차감

# 방법 2: Track Object 사용
track 10 interface serial 1/0.12 line-protocol
!
interface fastethernet 0/1
 standby 1 track 10 decrement 100
```

WAN 장애 시 G/W1의 Priority: 150 - 100 = **50** → G/W2(100)가 Active 승계

**예시 문제**: G/W1이 평소 Primary Gateway이고, G/W1의 내부(FastEthernet 0/1) 장애 시에는 G/W2가 즉시 승계, G/W1 복구 60초 후 다시 Active로 돌아와야 한다 — 여기까지는 `priority` + `preempt delay`로 해결된다. 그런데 G/W1의 **WAN 구간(Serial 1/0.12)**에 장애가 나도 HSRP는 이를 감지하지 못해 내부망에서는 여전히 G/W1이 Active로 남는 문제가 있다면?

```
# G/W1 — WAN 인터페이스를 track하여 장애 시 Priority를 100 낮춤
interface fastethernet 0/1
 standby 1 ip 200.20.2.254
 standby 1 priority 150
 standby 1 timer 1 3
 standby 1 preempt delay minimum 60      ! 복구 후 60초 뒤 Active 재취득
 standby 1 track serial 1/0.12 100       ! WAN 장애 시 Priority 150 - 100 = 50

# G/W2 — 상대적으로 우선순위가 높아지므로 preempt만 걸어두면 자동 승계
interface fastethernet 0/1
 standby 1 ip 200.20.2.254
 standby 1 priority 100
 standby 1 timer 1 3
 standby 1 preempt
```

확인: G/W1의 Serial 1/0.12를 `shutdown` 시키면 `G/W1# show standby all`에서 `Priority 50 (configured 150)`으로 표시되며 State가 Standby로, G/W2가 Active로 전환된다. `track`이 없으면 WAN이 끊겨도 LAN 쪽만 보는 HSRP는 이를 인지하지 못해 내부 PC들이 죽은 WAN 경로로 계속 트래픽을 보내는 블랙홀이 발생한다.

### 확인

```
G/W1# show standby brief
Interface  Grp  Pri  P  State    Active          Standby         Virtual IP
Fa0/1      1    150  P  Active   local           200.20.2.252    200.20.2.254

G/W2# show standby brief
Interface  Grp  Pri  P  State    Active          Standby         Virtual IP
Fa0/1      1    100     Standby  200.20.2.251    local           200.20.2.254

G/W1# show standby all
FastEthernet0/1 - Group 1
  State is Active
  Virtual IP address is 200.20.2.254
  Active virtual MAC address is 0000.0c07.ac01
  Hello time 1 sec, hold time 3 sec
  Priority 150 (configured 150)
    Track interface Serial1/0.12 state Up decrement 100
```

---

## HSRP 이중화 + 로드밸런싱 (Dual Group)

두 개의 HSRP 그룹으로 각 G/W가 서로의 Primary/Backup 역할을 교차 수행

```
         Group 1 Virtual IP: 200.20.2.254    ← PC1 그룹의 게이트웨이
         Group 2 Virtual IP: 200.20.2.253    ← PC2 그룹의 게이트웨이

G/W1: Group 1 Active, Group 2 Standby
G/W2: Group 1 Standby, Group 2 Active
```

```
# G/W1
interface fastethernet 0/1
 standby 1 ip 200.20.2.254
 standby 1 priority 150
 standby 1 timer 1 3
 standby 1 preempt delay minimum 60
 standby 1 track serial 1/0.12 100
 standby 2 ip 200.20.2.253
 standby 2 priority 100
 standby 2 timer 1 3
 standby 2 preempt

# G/W2
interface fastethernet 0/1
 standby 1 ip 200.20.2.254
 standby 1 priority 100
 standby 1 timer 1 3
 standby 1 preempt
 standby 2 ip 200.20.2.253
 standby 2 priority 150
 standby 2 timer 1 3
 standby 2 preempt delay minimum 60
 standby 2 track serial 1/0.23 100
```

---

## HSRP Version 2 (IOS XE 권장)

> 출처: [Cisco IOS XE 17.x HSRP v2 Configuration Guide](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ntw-servs/b-network-services/m_fhp-hsrp-v2.html)

### V1 vs V2 차이점

| 항목 | HSRP v1 | HSRP v2 |
|------|---------|---------|
| 그룹 번호 범위 | 0 ~ 255 | **0 ~ 4095** |
| 멀티캐스트 주소 | 224.0.0.2 | **224.0.0.102** |
| 밀리초 타이머 | 미지원 | **지원** |
| 가상 MAC | `0000.0c07.acXX` | `0000.0c9f.fXXX` |

### V2 설정

```
interface fastethernet 0/1
 standby version 2                      ! HSRP v2 활성화 (인터페이스 단위)
 standby 1 ip 200.20.2.254
 standby 1 priority 150
 standby 1 timer msec 200 msec 700      ! Hello 200ms, Hold 700ms (서브초 빠른 수렴)
 standby 1 preempt
```

> ⚠️ `standby version 2`는 인터페이스 단위로 설정. 같은 인터페이스의 모든 그룹에 적용됨

---

## VRRP (Virtual Router Redundancy Protocol)

HSRP를 표준화한 프로토콜 — 기본 동작 방식은 HSRP와 유사

| 항목 | HSRP | VRRP |
|------|------|------|
| 표준 | CISCO | IEEE |
| 용어 | Active/Standby | Master/Backup |
| Preempt 기본값 | 비활성 | **활성** |
| 가상 MAC | 0000.0c07.acXX | 0000.5e00.01XX |

### 기본 설정

```
# G/W1 (Master)
interface fastethernet 0/1
 vrrp 1 ip 200.20.2.254
 vrrp 1 priority 150
 vrrp 1 preempt delay minimum 60

# G/W2 (Backup)
interface fastethernet 0/1
 vrrp 1 ip 200.20.2.254
 vrrp 1 priority 100
```

### Track을 이용한 WAN 장애 감지

```
track 10 interface serial 1/0.12 line-protocol
!
interface fastethernet 0/1
 vrrp 1 ip 200.20.2.254
 vrrp 1 priority 150
 vrrp 1 preempt delay minimum 60
 vrrp 1 track 10 decrement 100
```

### 확인

```
G/W1# show vrrp brief
Interface  Grp  Pri  Time  Own  Pre  State   Master addr     Group addr
Fa0/1      1    150  3531       Y    Master  200.20.2.251    200.20.2.254

G/W1# show vrrp all
FastEthernet0/1 - Group 1
  State is Master
  Virtual IP address is 200.20.2.254
  Virtual MAC address is 0000.5e00.0101
  Priority is 150
  Preemption enabled, delay min 60 secs
  Track object 10 state Up decrement 100
```

---

## GLBP (Gateway Load Balancing Protocol)

- CISCO 전용
- **AVG (Active Virtual Gateway)**: 가상 IP 소유, MAC 할당 관리
- **AVF (Active Virtual Forwarder)**: 실제 트래픽을 포워딩하는 라우터
- 여러 AVF가 서로 다른 가상 MAC을 사용하여 **진정한 로드밸런싱** 실현

```
R1(config)# interface fastethernet 0/0
R1(config-if)# glbp 1 ip 192.168.10.254
R1(config-if)# glbp 1 priority 110
R1(config-if)# glbp 1 preempt
R1(config-if)# glbp 1 load-balancing round-robin
```

### GLBP 로드밸런싱 방식

| 방식 | 설명 |
|------|------|
| round-robin | 순차적으로 AVF에 MAC 할당 (기본값) |
| weighted | AVF 가중치에 따라 비율 조정 |
| host-dependent | 호스트 MAC 기반 고정 할당 |

---

## 정보 확인 (검증 명령어)

```
! HSRP 그룹 상세 — State(Active/Standby), 가상IP/MAC, 타이머, Priority, Track 상태
R# show standby

! HSRP 요약 — 인터페이스별 그룹 번호, Priority, State, Active/Standby 주소 한눈에 확인
R# show standby brief

! VRRP 상세 — Master/Backup State, 가상IP/MAC, Priority, Preempt 설정 확인
R# show vrrp

! VRRP 요약 — 그룹별 상태 빠르게 확인
R# show vrrp brief

! GLBP 상세 — AVG/AVF 역할, 가상IP, 로드밸런싱 방식 확인
R# show glbp

! GLBP 요약
R# show glbp brief

! Track 객체 상태 — Up/Down, 변경 횟수, 연결된 HSRP/VRRP 그룹 확인
R# show track
```

| 명령어 | 주로 확인하는 것 |
|--------|----------------|
| `show standby brief` | Active/Standby 역할 분배 현황 |
| `show standby` | Track 적용 후 Priority 감소 여부 확인 |
| `show track` | WAN 장애 감지 여부 (Up→Down 전환) |
| `show vrrp brief` | Master/Backup 역할, Preempt 활성 여부 |

---

## IOS XE 16.x / 17.x 최신 트렌드

### HSRP v2 + BFD 연동 (`IOS XE 16.x+`)

Track 기반 링크 감지보다 훨씬 빠른 장애 수렴 (sub-second)

```
! BFD 설정 (인터페이스)
interface GigabitEthernet0/1
 bfd interval 300 min_rx 300 multiplier 3   ! 약 900ms 이내 감지

! HSRP v2에 BFD 연동
interface GigabitEthernet0/1
 standby version 2
 standby 1 ip 192.168.10.254
 standby 1 priority 150
 standby 1 bfd                             ! BFD로 peer 감지
```

### VRRP v3 (IPv4 + IPv6 통합) (`IOS XE 16.x+`)

| 항목 | VRRP v2 (IOS 15.x) | VRRP v3 (IOS XE 16.x+) |
|------|--------------------|------------------------|
| IPv6 지원 | 별도 구성 | **통합 지원** |
| 타이머 | 초 단위 | **밀리초 단위** |
| 인증 | 미지원 (deprecated) | — |

```
! VRRP v3 설정 (IOS XE 16.x — IPv4/IPv6 통합)
fhrp version vrrp v3          ! 전역 v3 모드 활성화

interface GigabitEthernet0/1
 vrrp 1 address-family ipv4
  address 192.168.10.254 primary
  priority 150
  preempt delay minimum 30
  timers advertise msec 200   ! 200ms 광고 간격 (기본 1초)

 vrrp 2 address-family ipv6
  address FE80::1 primary
  address 2001:db8::1
  priority 150
```

### Enhanced Object Tracking (`IOS XE 16.x+`)

IP SLA 기반으로 단순 링크 상태가 아닌 **종단간 연결성** 추적 가능

```
! IP SLA로 인터넷 연결성 테스트 (8.8.8.8로 ICMP)
ip sla 1
 icmp-echo 8.8.8.8 source-interface GigabitEthernet0/1
 frequency 5              ! 5초마다 probe
ip sla schedule 1 start-time now life forever

! Track 객체 — IP SLA 기반
track 10 ip sla 1 reachability

! HSRP에 연동
interface GigabitEthernet0/1
 standby 1 track 10 decrement 60   ! 인터넷 끊기면 Priority -60
```

> IOS 15.x Phase 2에서는 `track X interface Y line-protocol`(링크 상태)만 사용.  
> IOS XE 16.x부터 `ip sla` 기반 종단간 연결성 추적으로 더 정교한 이중화 가능.
