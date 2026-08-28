# Inter-VLAN 라우팅

서로 다른 VLAN 간의 통신을 위해 Router가 필요하다.

---

## 방법 1: Sub-interface (Router on a Stick)

물리적 Interface 하나를 **다수의 논리적 Sub-interface**로 분할하여 사용

> 예: 3층 사무실에 VLAN 10(관리부), VLAN 20(개발부), VLAN 30(게스트)이 나뉘어 있고, Router의 물리 포트는 하나뿐인 상황에서 세 VLAN 간 통신이 가능해야 하는 경우

```
           Fa0/0 (Trunk)
R1 ─────────────────────── SW1
                              ├── VLAN 10 (192.168.10.x)
                              ├── VLAN 20 (192.168.20.x)
                              └── VLAN 30 (192.168.30.x)
```

### Router 설정

```
R1(config)# interface fastethernet 0/0
R1(config-if)# no shutdown
!
R1(config)# interface fastethernet 0/0.10
R1(config-subif)# encapsulation dot1q 10
R1(config-subif)# ip address 192.168.10.254 255.255.255.0
!
R1(config)# interface fastethernet 0/0.20
R1(config-subif)# encapsulation dot1q 20
R1(config-subif)# ip address 192.168.20.254 255.255.255.0
!
R1(config)# interface fastethernet 0/0.30
R1(config-subif)# encapsulation dot1q 30
R1(config-subif)# ip address 192.168.30.254 255.255.255.0
```

### Switch 설정 (Router 연결 포트를 Trunk로)

```
SW1(config)# interface fastethernet 0/24
SW1(config-if)# switchport trunk encapsulation dot1q
SW1(config-if)# switchport mode trunk
```

### 확인

```
R1# show ip route
C    192.168.10.0/24 is directly connected, FastEthernet0/0.10
C    192.168.20.0/24 is directly connected, FastEthernet0/0.20
C    192.168.30.0/24 is directly connected, FastEthernet0/0.30
```

---

## 방법 2: 물리적 Interface 분리

각 VLAN마다 Router의 별도 물리 Interface를 연결하는 방식
- 링크 수 = VLAN 수만큼 필요 → **비효율적**
- Sub-interface 방식이 일반적으로 선호됨

```
R1(config)# interface fastethernet 0/0
R1(config-if)# no shutdown
R1(config-if)# ip address 192.168.10.254 255.255.255.0
!
R1(config)# interface fastethernet 0/1
R1(config-if)# no shutdown
R1(config-if)# ip address 192.168.20.254 255.255.255.0
```

```
SW1(config)# interface fastethernet 0/22
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 10
!
SW1(config)# interface fastethernet 0/23
SW1(config-if)# switchport mode access
SW1(config-if)# switchport access vlan 20
```

---

> 💡 **Multi-Layer Switch**를 사용하면 Router 없이 스위치 자체에서 Inter-VLAN 라우팅 가능 → [Multi-Layer Switch](Multi-Layer-Switch) 참고

---

## IOS XE 16.x / 17.x 최신 트렌드

### SVI (Switched Virtual Interface) 방식 — 권장 (`IOS XE 16.x+`)

Phase 2 자료의 **Router-on-a-Stick(Sub-interface)** 방식 대신, L3 스위치 SVI가 현재 표준.

| 방식 | IOS 15.x Phase 2 | IOS XE 16.x+ 권장 |
|------|-----------------|-------------------|
| 구성 | Router Sub-interface | L3 Switch SVI |
| 장비 | Router + Switch 분리 | L3 Switch 단독 |
| 성능 | Router CPU 처리 | **하드웨어 ASIC 처리** |
| 확장성 | 제한적 | 우수 |

```
! IOS XE 16.x L3 Switch SVI 방식 (권장)
SW(config)# ip routing                          ! L3 라우팅 활성화

SW(config)# interface vlan 10
SW(config-if)# ip address 192.168.10.1 255.255.255.0
SW(config-if)# no shutdown

SW(config)# interface vlan 20
SW(config-if)# ip address 192.168.20.1 255.255.255.0
SW(config-if)# no shutdown

! 외부 인터넷 연결 — Routed Port (IOS XE L3 Switch)
SW(config)# interface GigabitEthernet0/1
SW(config-if)# no switchport               ! L3 포트로 전환
SW(config-if)# ip address 203.0.113.2 255.255.255.252
SW(config-if)# no shutdown

SW(config)# ip route 0.0.0.0 0.0.0.0 203.0.113.1   ! Default Route
```

```
! 확인
SW# show ip route                  ! VLAN 간 라우팅 경로 확인
SW# show interface vlan 10         ! SVI 상태 확인
SW# show ip interface brief        ! 전체 L3 인터페이스 요약
```

> Sub-interface 방식은 여전히 동작하지만, IOS XE 16.x 이상 환경에서는 SVI 방식이 성능·확장성·관리 측면에서 우세.
