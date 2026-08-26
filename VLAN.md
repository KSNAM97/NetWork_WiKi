# VLAN (Virtual LAN)

## 개념

- Switch는 Collision Domain을 분할하지만 **Broadcast Domain을 공유**한다.
- Host 수 증가 → ARP, DHCP 등 Broadcast 트래픽 증가 → 네트워크 효율 저하
- VLAN을 사용하면 **물리적 분할 없이 논리적으로 네트워크를 분할** 가능

**장점**
- Broadcast Traffic 분할로 효율적인 LAN 구성
- 위치·물리적 제한 없이 네트워크 분리 가능

---

## VLAN 번호 체계

- VLAN은 **12bit** 구성 (0 ~ 4095)
- 0, 4095는 시스템 예약 → 사용 가능 범위: **1 ~ 4094**

| 종류 | 범위 | 비고 |
|------|------|------|
| Standard VLAN | 1 ~ 1005 | 1, 1002~1005는 예약 VLAN |
| Extended VLAN | 1006 ~ 4094 | 장비 시리즈/IOS에 따라 지원 여부 다름 |
| Default VLAN | 1 | 모든 포트가 기본적으로 속하는 VLAN |

---

## VLAN 생성 · 수정 · 삭제

### 단일 VLAN 생성

```
Switch(config)# vlan 10
```

### 다중 VLAN 생성

```
Switch(config)# vlan 10,20,30
Switch(config)# vlan 11-30
```

### VLAN 이름 변경

```
Switch(config)# vlan 10
Switch(config-vlan)# name CISCO-CCNA
```

### VLAN 삭제

```
Switch(config)# no vlan 10
Switch(config)# no vlan 10,20,30
Switch(config)# no vlan 11-30
```

### 정보 확인

```
Switch# show vlan brief
```

---

## 설정 예시

```
SW1(config)# vlan 11
SW1(config-vlan)# name CISCO-CCNA
!
SW1(config)# vlan 12
SW1(config-vlan)# name CISCO-CCNP
!
SW1(config)# vlan 13
SW1(config-vlan)# name LINUX-Server
!
SW1(config)# vlan 14
SW1(config-vlan)# name MS-Server
!
SW1(config)# vlan 15
SW1(config-vlan)# name AWS
```

---

## IOS XE 16.x / 17.x 최신 트렌드

### VLAN 확인 명령어 강화 (`IOS XE 16.x+`)

기본 VLAN 생성/삭제 문법은 IOS 15.x와 동일. IOS XE 16.x부터 확인 명령어에 필터링 옵션 추가.

```
SW# show vlan brief
SW# show vlan id 10                 ! 특정 VLAN만 확인
SW# show vlan name SALES            ! VLAN 이름으로 검색
```

### Private VLAN (`IOS XE 16.x — Catalyst 기준`)

같은 VLAN 내 호스트 간 통신을 격리하는 기능 — 보안 강화 목적

| PVLAN 타입 | 설명 |
|-----------|------|
| **Primary** | 외부 통신용 VLAN |
| **Isolated** | 포트 간 상호 통신 완전 차단 |
| **Community** | 같은 Community 내 통신만 허용 |

```
! Private VLAN 설정 (IOS XE 16.x+)
! 1단계: Secondary VLAN 생성
SW(config)# vlan 101
SW(config-vlan)# private-vlan isolated      ! 격리 VLAN

SW(config)# vlan 102
SW(config-vlan)# private-vlan community     ! 커뮤니티 VLAN

! 2단계: Primary VLAN과 연결
SW(config)# vlan 100
SW(config-vlan)# private-vlan primary
SW(config-vlan)# private-vlan association 101,102

! 3단계: 포트 타입 지정
SW(config)# interface GigabitEthernet0/1
SW(config-if)# switchport mode private-vlan host
SW(config-if)# switchport private-vlan host-association 100 101   ! Primary 100, Secondary 101
```

```
! 확인
SW# show vlan private-vlan
SW# show interface GigabitEthernet0/1 switchport
```

> IOS 15.x Phase 2 자료에는 Private VLAN 미포함. IOS XE 16.x 이상 Catalyst 스위치에서 지원.
