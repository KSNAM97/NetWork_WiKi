# Spanning Tree Protocol (STP)

스위치 네트워크에서 **루프(Loop)를 방지**하기 위한 프로토콜

---

## 기본 개념

- 물리적 루프가 있는 스위치 환경에서 **논리적으로 특정 포트를 차단**하여 루프 방지
- 차단된 포트는 장애 발생 시 자동으로 활성화 → **자동 복구**

---

## STP 포트 상태 (IEEE 802.1D)

| 상태 | 설명 | MAC 학습 | 트래픽 전달 |
|------|------|----------|------------|
| **Blocking** | 루프 방지를 위해 차단 | X | X |
| **Listening** | 루트 포트/지정 포트 선출 | X | X |
| **Learning** | MAC Address 학습 | O | X |
| **Forwarding** | 정상 동작 | O | O |
| **Disabled** | 관리자가 비활성화 | X | X |

> **Blocking → Forwarding 전환 시간**: 기본 30~50초

---

## Root Bridge 선출

### 선출 기준

1. **Bridge Priority** — 낮을수록 유리 (기본값: 32768)
2. **MAC Address** — 낮을수록 유리 (Priority가 동일한 경우)

### Root Bridge 설정

```
SW(config)# spanning-tree vlan [VLAN번호] priority [0/4096/8192/.../61440]
SW(config)# spanning-tree vlan [VLAN번호] root primary      ← 자동으로 가장 낮은 값 설정
SW(config)# spanning-tree vlan [VLAN번호] root secondary    ← BridgePriority = 28672
```

---

## 포트 역할

| 역할 | 설명 |
|------|------|
| **Root Port** | Root Bridge까지 최단 경로 포트 |
| **Designated Port** | 각 세그먼트에서 Root Bridge 방향으로 트래픽 전달 포트 |
| **Non-Designated Port** | 루프 방지를 위해 차단되는 포트 |

---

## Cost (경로 비용)

| 속도 | STP Cost |
|------|----------|
| 10 Gbps | 2 |
| 1 Gbps | 4 |
| 100 Mbps | 19 |
| 10 Mbps | 100 |

### Cost 조정

```
SW(config-if)# spanning-tree vlan [VLAN] cost [값]
```

### Port Priority 조정

```
SW(config-if)# spanning-tree vlan [VLAN] port-priority [0~240, 기본 128]
```

---

## PVST+ (Per-VLAN Spanning Tree)

- CISCO 전용
- **VLAN별로 독립적인 STP 인스턴스** 실행
- VLAN마다 다른 Root Bridge 설정 가능 → 로드 밸런싱

---

## RSTP (IEEE 802.1W — Rapid PVST+)

| 항목 | STP (802.1D) | RSTP (802.1W) |
|------|-------------|----------------|
| 수렴 시간 | 30~50초 | **1~2초** |
| 포트 상태 수 | 5개 | 3개 |

### RSTP 포트 상태

| 상태 | 설명 |
|------|------|
| Discarding | Blocking + Listening 통합 |
| Learning | MAC 학습 |
| Forwarding | 정상 동작 |

```
SW(config)# spanning-tree mode rapid-pvst
```

---

## MSTP (IEEE 802.1S — Multiple Spanning Tree)

- 여러 VLAN을 하나의 **인스턴스(Instance)**로 묶어 STP 실행
- RSTP 기반으로 동작
- 대규모 환경에서 STP 인스턴스 수 절감

```
SW(config)# spanning-tree mode mst
SW(config)# spanning-tree mst configuration
SW(config-mst)# name [이름]
SW(config-mst)# revision [번호]
SW(config-mst)# instance 1 vlan 10,20,30
SW(config-mst)# instance 2 vlan 40,50,60
```

---

## STP 보호 기술

### PortFast

- **Access 포트**에서 STP 계산 없이 즉시 Forwarding 상태로 전환
- PC, Server 연결 포트에 적용

```
SW(config-if)# spanning-tree portfast
```

### BPDU Guard

- PortFast 포트에서 BPDU 수신 시 포트를 **err-disable 상태**로 전환
- 엔드 디바이스 포트에 스위치가 연결되는 것을 방지

```
SW(config-if)# spanning-tree bpduguard enable
SW(config)# spanning-tree portfast bpduguard default    ← 전체 PortFast 포트에 적용
```

### Root Guard

- 특정 포트에서 **Root Bridge 역할을 하는 BPDU가 수신되면 차단**
- Root Bridge 위치 보호

```
SW(config-if)# spanning-tree guard root
```

### Loop Guard

- 단방향 링크 장애 시 Non-Designated 포트가 Forwarding 전환하는 것을 방지

```
SW(config-if)# spanning-tree guard loop
```

---

## 정보 확인

```
SW# show spanning-tree
SW# show spanning-tree vlan [번호]
SW# show spanning-tree summary
```

---

## IOS XE 16.x / 17.x 최신 트렌드

### RSTP (802.1w) — IOS XE 기본값 (`IOS XE 16.x+`)

IOS XE 16.x 이상에서는 **RSTP가 기본 STP 모드**. IOS 15.x Phase 2 자료의 Classic STP(802.1d) 대비 수렴 속도 대폭 향상.

| 항목 | Classic STP (IOS 15.x) | RSTP / MSTP (IOS XE 16.x+) |
|------|------------------------|----------------------------|
| 수렴 시간 | 30~50초 | **1~2초** |
| 포트 상태 | Blocking/Listening/Learning/Forwarding | Discarding/Learning/Forwarding |
| Proposal/Agreement | 없음 | **있음 (빠른 전환)** |
| 기본 모드 | STP (pvst) | **rapid-pvst** |

```
! IOS XE 16.x 기본 STP 모드 확인 및 변경
SW# show spanning-tree summary
!  Switch is in rapid-pvst mode   ← IOS XE 기본값

! MSTP (802.1s) 설정 — 여러 VLAN을 Instance로 묶어 STP 부하 감소
SW(config)# spanning-tree mode mst

SW(config)# spanning-tree mst configuration
SW(config-mst)# name CAMPUS_MST
SW(config-mst)# revision 1
SW(config-mst)# instance 1 vlan 10,20,30    ! Instance 1 = VLAN 10,20,30
SW(config-mst)# instance 2 vlan 40,50,60    ! Instance 2 = VLAN 40,50,60

SW(config)# spanning-tree mst 1 priority 4096   ! Instance 1 Root Bridge
SW(config)# spanning-tree mst 2 priority 8192   ! Instance 2 Root Bridge
```

### STP 보호 기술 (`IOS XE 16.x+`)

```
! BPDU Guard — PortFast 포트에서 BPDU 수신 시 즉시 err-disable
SW(config)# spanning-tree portfast bpduguard default    ! 전역 적용

! Root Guard — 특정 포트에서 Superior BPDU 수신 시 Root-inconsistent 차단
SW(config-if)# spanning-tree guard root

! Loop Guard — 단방향 링크 장애로 인한 루프 방지
SW(config)# spanning-tree loopguard default             ! 전역 적용
SW(config-if)# spanning-tree guard loop                ! 인터페이스별

! BPDU Filter — BPDU 송수신 완전 차단 (주의: 루프 위험)
SW(config-if)# spanning-tree bpdufilter enable
```

```
! 확인 (IOS XE 16.x+)
SW# show spanning-tree summary                  ! STP 모드, PortFast/Guard 현황
SW# show spanning-tree vlan 10 detail           ! 포트별 Role/State 상세
SW# show spanning-tree inconsistentports        ! Root Guard/Loop Guard 차단 포트
SW# show spanning-tree mst                      ! MSTP Instance 정보
```
