# Switchport Mode

## Access Mode

- 하나의 Switchport로 **하나의 VLAN만** 사용하여 통신
- PC, Server 등 **단말 장비**가 연결되는 포트
- VLAN을 지원하지 않는 장비가 연결되는 포트

```
SW(config)# interface fastethernet 0/1
SW(config-if)# switchport mode access
SW(config-if)# switchport access vlan 10
```

---

## Trunk Mode

- 하나의 Switchport로 **복수 개의 VLAN**을 사용하여 통신
- Switch, IP Phone 등 VLAN을 지원하는 장비가 연결되는 포트

```
SW(config)# interface fastethernet 0/24
SW(config-if)# switchport trunk encapsulation dot1q
SW(config-if)# switchport mode trunk
```

---

## Dynamic Mode (DTP)

DTP(Dynamic Trunking Protocol) 메시지를 사용하여 Switch 간 협상으로 Trunk 구성

| 모드 | 동작 |
|------|------|
| `desirable` | DTP 메시지 **송수신** — 적극적으로 Trunk 협상 |
| `auto` | DTP 메시지 **수신만** — 상대방이 desirable일 때만 Trunk |

```
SW(config)# interface fastethernet 0/1
SW(config-if)# switchport mode dynamic desirable
```

### Dynamic 모드 협상 결과

|  | trunk | desirable | auto | access |
|--|-------|-----------|------|--------|
| **trunk** | Trunk | Trunk | Trunk | X |
| **desirable** | Trunk | Trunk | Trunk | X |
| **auto** | Trunk | Trunk | Access | Access |
| **access** | X | X | Access | Access |

---

## Range Command

동일한 설정을 여러 Interface에 한 번에 적용

```
SW(config)# interface range fa0/1 - 8
SW(config-if-range)# switchport mode access
SW(config-if-range)# switchport access vlan 10

SW(config)# interface range fa0/13, fa0/15, fa0/17, fa0/19
SW(config-if-range)# switchport mode access
SW(config-if-range)# switchport access vlan 20
```

---

## 정보 확인

```
Switch# show vlan brief
Switch# show interface trunk
Switch# show interface fa0/1 switchport
```

---

## IOS XE 16.x / 17.x 최신 트렌드

### 인터페이스 범위 설정 강화 (`IOS XE 16.x+`)

```
! IOS 15.x — 범위 지정 방식
SW(config)# interface range fa0/1 - 10

! IOS XE 16.x — GigabitEthernet 기본, 범위 표현 유연
SW(config)# interface range GigabitEthernet1/0/1 - 24
SW(config)# interface range Gi1/0/1 - 12, Gi1/0/20 - 24   ! 불연속 범위 동시 지정
```

### 802.1X 기반 Dynamic VLAN 할당 (`IOS XE 16.x+`)

Switchport Mode를 수동 설정하지 않고 **RADIUS 서버에서 VLAN을 동적 할당**하는 방식 — 기업 환경 표준

```
! AAA 및 802.1X 설정 (IOS XE 16.x+)
aaa new-model
aaa authentication dot1x default group radius
aaa authorization network default group radius

! 전역 802.1X 활성화
dot1x system-auth-control

! 포트별 설정
interface GigabitEthernet1/0/1
 switchport mode access
 authentication port-control auto     ! 802.1X 인증 요구
 dot1x pae authenticator
 authentication host-mode multi-auth  ! 여러 클라이언트 인증 허용

! RADIUS 서버 지정
radius server ISE
 address ipv4 192.168.1.100 auth-port 1812 acct-port 1813
 key RADIUS_SECRET
```

> IOS 15.x Phase 2 자료에서는 수동 `switchport access vlan X` 방식만 사용.  
> IOS XE 16.x 기업 환경에서는 802.1X + ISE(Identity Services Engine) 연동으로 VLAN 자동 할당.
