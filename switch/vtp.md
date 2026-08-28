# VTP (VLAN Trunking Protocol)

- CISCO 전용 프로토콜
- Trunk 링크를 통해 **VLAN 정보를 자동으로 공유**하는 프로토콜
- VLAN 설정을 한 곳(Server)에서 관리하여 일관성 유지

---

## VTP 사용 조건

VTP로 VLAN 정보를 동기화하려면 세 가지 조건이 모두 맞아야 한다.

1. **Trunk 연결** — Switch간 반드시 Trunk로 연결되어야 함 (Access 링크로는 VTP 광고가 전달되지 않음)
2. **VTP Domain 일치** — Domain이 다르면 그 Switch를 기준으로 양쪽은 VLAN을 동기화하지 않음. Domain은 각 Switch에서 개별 설정하지만, Trunk로 연결된 상태에서 한쪽에서 설정하면 자동으로 전파됨
3. **VTP Password 일치** — Password는 전파되지 않으므로 각 Switch에서 개별적으로 동일하게 설정해야 함

---

## VTP 모드

| 모드 | VLAN 생성 | VLAN 수정 | VLAN 삭제 | 전파 | 일치 | 중계 | 저장 |
|------|-----------|-----------|-----------|------|------|------|------|
| **Server** (기본값) | O | O | O | O | O | O | O (NVRAM) |
| **Client** | X | X | X | X | O | O | X |
| **Transparent** | O (로컬만) | O (로컬만) | O (로컬만) | X | X | O | O (로컬만) |

- **전파(Advertise)**: 자신의 VLAN 변경 정보를 Trunk로 연결된 다른 Switch에 동기화시키는 권한
- **일치(Synchronize)**: Trunk로부터 받은 VLAN 변경 정보를 자신에게 반영하는 권한
- **중계(Relay)**: Trunk로부터 받은 VLAN 정보를 다른 Trunk 포트로 그대로 전달하는 권한 — Transparent 모드도 자신은 동기화하지 않지만 중계는 수행

> Revision Number는 전파 또는 일치 중 최소 하나의 권한이 있어야 증가한다.

---

### Extended VLAN과 VTP mode

> 조건: SW1에서 VLAN 3001(Extended VLAN)을 생성해야 한다. 현재 SW1은 VTP Server 모드다.

Server/Client 모드에서는 **Extended VLAN(1006-4094)이 생성되지 않는다.** VLAN 3001을 만들려면 먼저 Transparent 모드로 전환해야 한다.

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

## VTP 광고 메시지 종류

VTP는 VLAN 1을 통해 아래 세 종류의 메시지를 주고받으며 VLAN Database를 동기화한다.

| 메시지 | 발신 | 설명 |
|--------|------|------|
| **Summary-Advertisement** | VTP Server | VLAN Database 변경 시 Revision Number를 전파. 변경이 없어도 5분마다 주기적으로 전파 |
| **Advertise-Request** | 자신보다 낮은 Revision을 가진 Switch | Summary-Advertisement 수신 후 자신보다 Revision이 높음을 확인하면 상세 정보를 요청 |
| **Subset-Advertisement** | 요청받은 Switch | Advertise-Request에 대한 응답으로 변경된 VLAN Database 상세 정보를 전달 |

---

### 혼합 모드(Server-Client-Transparent-Client) 구성 예시

> 조건: SW1(Server)만 VLAN을 생성, SW2(Client)·SW4(Client)는 SW1로부터만 VLAN을 공유, SW3(Transparent)은 다른 Switch와 VLAN 정보를 공유하지 않는다. VTP Domain = GIT_NET, Password = cisco1234.

```
   VTP Server        VTP Client        VTP Transparent        VTP Client
  SW1 ------------- SW2 ------------- SW3 ------------------- SW4
      Fa0/20             Fa0/21               Fa0/22
```

```
# SW1
interface fastethernet 0/20
 switchport trunk encapsulation dot1q
 switchport mode trunk
!
vtp mode server
vtp domain GIT_NET
vtp password cisco1234

# SW2
interface range fa0/20 - 21
 switchport trunk encapsulation dot1q
 switchport mode trunk
!
vtp mode client
vtp domain GIT_NET
vtp password cisco1234

# SW3 (Transparent — VLAN을 로컬로만 생성/관리, SW2↔SW4 사이는 그대로 중계만 수행)
interface range fa0/21 - 22
 switchport trunk encapsulation dot1q
 switchport mode trunk
!
vtp mode transparent
vtp domain GIT_NET
vtp password cisco1234

# SW4
interface fastethernet 0/22
 switchport trunk encapsulation dot1q
 switchport mode trunk
!
vtp mode client
vtp domain GIT_NET
vtp password cisco1234
```

SW1에서 생성한 VLAN은 SW2까지만 동기화되고, SW3은 Transparent이므로 자신의 VLAN Database에는 반영하지 않지만 Trunk 프레임 자체는 SW4로 중계한다 — 단, SW4는 SW3의 VLAN 변경 정보를 받는 것이 아니라 SW1이 원래 SW2 경유로 보낸 광고를 SW3이 그대로 전달해주는 것이므로, SW3에서 직접 VLAN을 만들어도 SW2·SW4에는 반영되지 않는다.

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
