# VTP (VLAN Trunking Protocol)

- CISCO 전용 프로토콜
- Trunk 링크를 통해 **VLAN 정보를 자동으로 공유**하는 프로토콜
- VLAN 설정을 한 곳(Server)에서 관리하여 일관성 유지

---

## VTP 모드

| 모드 | VLAN 생성/삭제 | 광고 수신 | 광고 전달 | 저장 |
|------|---------------|-----------|-----------|------|
| **Server** | O | O | O | O (NVRAM) |
| **Client** | X | O | O | X |
| **Transparent** | O (로컬만) | O (무시) | O | O (로컬만) |

---

### Extended VLAN과 VTP mode

> 조건: SW1에서 VLAN 3001(Extended VLAN)을 생성해야 한다. 현재 SW1은 VTP Server 모드다.

Server/Client 모드에서는 **Extended VLAN(1006~4094)이 생성되지 않는다.** VLAN 3001을 만들려면 먼저 Transparent 모드로 전환해야 한다.

```
SW1(config)# vtp mode transparent
SW1(config)# vlan 3001
SW1(config-vlan)# exit
```

- Extended VLAN은 로컬(Transparent)에만 저장되며 다른 스위치로 동기화되지 않는다.
- SW1을 다시 `vtp mode server`로 되돌리려면 생성해둔 Extended VLAN(3001)을 먼저 삭제해야 한다. 삭제하지 않으면 Server 모드 전환 자체가 거부된다.

---

## 기본 설정

```
SW(config)# vtp mode [server | client | transparent]
SW(config)# vtp domain [도메인명]
SW(config)# vtp password [비밀번호]
SW(config)# vtp version [1 | 2 | 3]
```

---

## VTP Revision 번호

- VLAN 변경 시마다 **Revision 번호 +1 증가**
- 스위치들은 더 높은 Revision 번호를 가진 스위치의 VLAN 정보를 적용
- ⚠️ **높은 Revision 번호의 스위치를 네트워크에 추가 시 기존 VLAN 정보 덮어씌워짐**

### Revision 번호 초기화

```
SW(config)# vtp mode transparent
SW(config)# vtp mode server       ← 다시 Server로 변경 시 Revision 0으로 초기화
```

---

## VTP Pruning

- Trunk 링크를 통해 불필요한 VLAN 트래픽을 차단하여 **대역폭 절약**
- Server에서 활성화 시 전체 도메인에 적용

```
SW(config)# vtp pruning
```

---

## 설정 예시

```
# VTP Server (SW1)
SW1(config)# vtp mode server
SW1(config)# vtp domain CISCO_LAB
SW1(config)# vtp password cisco123
SW1(config)# vlan 10
SW1(config-vlan)# name SALES
SW1(config)# vlan 20
SW1(config-vlan)# name ENGINEER

# VTP Client (SW2, SW3)
SW2(config)# vtp mode client
SW2(config)# vtp domain CISCO_LAB
SW2(config)# vtp password cisco123
```

SW1에서 VLAN을 추가/삭제하면 SW2, SW3에도 자동으로 반영됨

### 실습 예시: 여러 Server가 VLAN을 나눠서 생성

> 조건: SW1, SW2, SW3 모두 VTP Domain = Global_IT, VTP Password = GIT_cisco 로 Server 모드다. SW1에서 VLAN 11-13, SW2에서 VLAN 21-23, SW3에서 VLAN 31-33을 각각 생성하면, 세 스위치 모두에서 VLAN 11-13, 21-23, 31-33이 전부 조회되어야 한다.

```
# SW1
SW1(config)# vtp mode server
SW1(config)# vtp domain Global_IT
SW1(config)# vtp password GIT_cisco
SW1(config)# vlan 11-13

# SW2
SW2(config)# vtp mode server
SW2(config)# vtp domain Global_IT
SW2(config)# vtp password GIT_cisco
SW2(config)# vlan 21-23

# SW3
SW3(config)# vtp mode server
SW3(config)# vtp domain Global_IT
SW3(config)# vtp password GIT_cisco
SW3(config)# vlan 31-33
```

Domain과 Password가 동일하고 Trunk로 연결되어 있으면, Server 모드끼리도 서로의 VLAN 광고를 주고받아 세 스위치 모두 9개 VLAN을 동일하게 갖게 된다. `show vtp status`로 Revision 번호가, `show vlan brief`로 VLAN 목록이 일치하는지 확인한다.

---

## 정보 확인

```
SW# show vtp status
SW# show vtp counters
SW# show vlan brief
```

---

## IOS XE 16.x / 17.x 최신 트렌드

### VTP Version 3 (`IOS XE 16.x+`)

Phase 2 자료까지는 VTPv1/v2가 기준. IOS XE 16.x 이상 Catalyst에서 **VTPv3**가 표준.

| 항목 | VTPv1/v2 (IOS 15.x) | VTPv3 (IOS XE 16.x+) |
|------|---------------------|----------------------|
| Primary Server | 개념 없음 (Server 모드면 전파) | **Primary Server 명시 지정** |
| VLAN 범위 | 1-1005 | **1-4094 (Extended VLAN 포함)** |
| Private VLAN | 미지원 | **지원** |
| MST 정보 전파 | 미지원 | **지원** |
| 패스워드 암호화 | 평문 | **암호화 저장** |

```
! VTPv3 설정 (IOS XE 16.x+)
SW(config)# vtp version 3
SW(config)# vtp domain CISCO_LAB
SW(config)# vtp password cisco123 hidden    ! 암호화 저장

! Primary Server 지정 — VTPv3는 반드시 Primary Server를 수동 활성화해야 전파
SW# vtp primary                             ! Privileged mode에서 실행
! → "This system is becoming primary server for feature vlan" 확인

! Client 스위치
SW2(config)# vtp version 3
SW2(config)# vtp mode client
SW2(config)# vtp domain CISCO_LAB
SW2(config)# vtp password cisco123 hidden
```

```
! 확인 (VTPv3 추가 항목)
SW# show vtp status
!  VTP Version capable             : 1 to 3
!  VTP version running             : 3
!  VTP Primary Server              : Yes   ← Primary 여부 표시
SW# show vtp password              ! 암호화된 패스워드 확인
```

> VTPv3에서는 Primary Server가 아닌 Server 모드 스위치는 VLAN 전파 불가.  
> IOS 15.x Phase 2 자료의 VTPv2와 Primary Server 개념이 다름 — 혼용 주의.
