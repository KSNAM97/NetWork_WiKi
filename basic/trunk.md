# Trunk

## 개념

- Switch 간 연결 시 Access Mode는 VLAN 수만큼 링크가 필요 → 비효율적
- **Trunk Mode**: 하나의 링크로 다수의 VLAN 트래픽을 전달

---

## Trunk 방식

| 방식 | 설명 |
|------|------|
| **IEEE 802.1Q** | IEEE 표준 Trunk, Native VLAN 지원 |
| **ISL** | CISCO 전용 Trunk 프로토콜, Native VLAN 미지원 |

> CISCO 2950 이하 장비는 ISL을 지원하지 않으며 `dot1q`만 사용 가능

---

## 설정

### IEEE 802.1Q

```
SW(config)# interface fastethernet 0/24
SW(config-if)# switchport trunk encapsulation dot1q
SW(config-if)# switchport mode trunk
```

### ISL

```
SW(config)# interface fastethernet 0/24
SW(config-if)# switchport trunk encapsulation isl
SW(config-if)# switchport mode trunk
```

### 2950 이하 (ISL 미지원)

```
SW(config)# interface fastethernet 0/24
SW(config-if)# switchport mode trunk
```

---

## IEEE 802.1Q Tagging

Access Mode와 달리 Trunk Mode에서는 Ethernet Frame에 **4Byte Tag**가 추가된다.

**Access Mode Ethernet Header**

```
| Dst MAC | Src MAC | Type | Data |
```

**Trunk Mode Ethernet Header (802.1Q)**

```
| Dst MAC | Src MAC | Tagging(4B) | Type | Data |
```

**4Byte (32bit) 구성**

| 필드 | 크기 | 설명 |
|------|------|------|
| EtherType | 16bit | `0x8100` — 802.1Q임을 표기 |
| Priority | 3bit | QoS 우선순위 (8단계) |
| CFI | 1bit | 0=Ethernet, 1=Token-ring |
| VLAN-ID | 12bit | VLAN 번호 (0-4095) |

---

## Native VLAN

- **802.1Q**: VLAN 태그가 없는 트래픽을 **Native VLAN(기본 VLAN 1)**으로 처리
- **ISL**: 태그가 없는 트래픽을 자신의 것이 아닌 것으로 간주하여 **Drop**

### Native VLAN 변경

```
SW(config-if)# switchport trunk native vlan 10
```

> 예: SW1과 SW2 사이 Gi1/0/1 링크 하나로 VLAN 10(영업팀), VLAN 20(개발팀) 트래픽만 통과시키고 Native VLAN은 기본값 1이 아닌 999로 변경해야 하는 경우

```
SW1(config)# interface gigabitEthernet 1/0/1
SW1(config-if)# switchport mode trunk
SW1(config-if)# switchport trunk native vlan 999
SW1(config-if)# switchport trunk allowed vlan 10,20
```

---

## 정보 확인

```
SW# show interface trunk
SW# show interface fa0/24 trunk
SW# show dtp interface fa0/24
```

| 명령어 | 주로 확인하는 것 |
|--------|----------------|
| `show interface trunk` | Trunk 포트 전체 목록 및 허용 VLAN |
| `show interface X trunk` | 해당 포트의 Active/Forwarding VLAN 필터링 결과 |
| `show dtp interface X` | DTP 비활성화(`nonegotiate`) 적용 여부 |

---

## IOS XE 16.x / 17.x 최신 트렌드

### Trunk 기본 동작 변화 (`IOS XE 16.x+`)

Catalyst 9000은 ISL 미지원 — `switchport trunk encapsulation dot1q` 명령어 없음. GigabitEthernet이 기본 포트.

```
! IOS 15.x — FastEthernet 위주, encapsulation 명시 필요
interface FastEthernet0/24
 switchport trunk encapsulation dot1q
 switchport mode trunk

! IOS XE 16.x+ Catalyst 9000 — encapsulation 불필요 (dot1q 고정)
interface GigabitEthernet1/0/24
 switchport mode trunk
 switchport nonegotiate             ! DTP 협상 중단 — VLAN Hopping 방어 (보안 권장)
 switchport trunk native vlan 999   ! VLAN 1 회피
 switchport trunk allowed vlan 10,20,30
```

### Allowed VLAN 제어

```
SW(config-if)# switchport trunk allowed vlan 10,20,30
SW(config-if)# switchport trunk allowed vlan add 40,50
SW(config-if)# switchport trunk allowed vlan remove 5-10,12
SW(config)# vtp pruning    ! 불필요한 VLAN 트래픽 자동 차단 (VTP Server에서 활성화)
```

### QinQ (802.1ad) — Double Tagging (`IOS XE 17.x`)

서비스 프로바이더 환경에서 고객 VLAN 태그 위에 Provider 태그를 추가.

```
interface GigabitEthernet1/0/1
 switchport access vlan 100
 switchport mode dot1q-tunnel
 l2protocol-tunnel cdp
```

| 버전 | Trunk 특이사항 |
|------|--------------|
| IOS 15.x (Phase 2) | ISL/dot1q 선택, DTP 기본 활성, FastEthernet 위주 |
| IOS XE 16.x+ | dot1q 고정(ISL 미지원), nonegotiate 권장, GigabitEthernet 기본 |
| IOS XE 17.x | QinQ(802.1ad) 지원, MACsec Trunk 암호화 가능 |
