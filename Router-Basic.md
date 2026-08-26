# Router 기본

---

## Router 동작 원리

- Layer 3 장비 — IP Header의 Destination IP를 기반으로 Routing Table 조회 후 Next-hop으로 전달
- Routing Table에 없는 목적지 패킷은 Drop
- 직접 연결된 네트워크(Connected)는 자동으로 Routing Table에 등록
- 원격 네트워크는 Static 또는 Dynamic Routing Protocol을 통해 등록

---

## Router 모드

| 모드 | 프롬프트 | 설명 |
|------|---------|------|
| User Mode | `Router>` | 기본 접속 모드 — 제한된 show 명령어만 사용 가능 |
| Privilege Mode | `Router#` | 전체 show/debug/copy 명령어 사용 가능 |
| Global Mode | `Router(config)#` | 설정 입력 — RAM에 즉시 반영 |

```
Router> enable                ← User → Privilege
Router# configure terminal    ← Privilege → Global
Router(config)# exit          ← 이전 모드로 복귀
```

---

## RAM / NVRAM

|  | RAM | NVRAM |
|--|-----|-------|
| 특징 | 휘발성 메모리 | 비휘발성 메모리 |
| 확인 | `show running-config` | `show startup-config` |
| 저장 | 설정 입력 시 자동 반영 | `copy running-config startup-config` |
| 삭제 | `reload` | `erase startup-config` |

---

## Interface 설정

### Ethernet Interface (LAN)

```
R1(config)# interface fastethernet 0/0
R1(config-if)# no shutdown
R1(config-if)# ip address 192.168.1.254 255.255.255.0
```

### Serial Interface (WAN)

- WAN 구간 연결용 — Layer 2 Protocol: HDLC (기본값)
- DCE 측에서 `clock rate` 설정 필요

```
! DTE 측
R1(config)# interface serial 1/0
R1(config-if)# no shutdown
R1(config-if)# encapsulation hdlc
R1(config-if)# bandwidth 64
R1(config-if)# ip address 192.168.12.1 255.255.255.0

! DCE 측
R2(config)# interface serial 1/1
R2(config-if)# no shutdown
R2(config-if)# encapsulation hdlc
R2(config-if)# bandwidth 64
R2(config-if)# clock rate 64000
R2(config-if)# ip address 192.168.12.2 255.255.255.0
```

### Loopback Interface (가상)

- 물리적 포트 없이 항상 Up 상태 유지 — 테스트/라우팅 ID 용도

```
Router(config)# interface loopback 0
Router(config-if)# ip address 1.1.1.1 255.255.255.255
```

### Interface 상태 확인

```
R# show ip interface brief
Interface         IP-Address      OK? Method  Status                Protocol
Serial1/0         unassigned      YES unset   administratively down down  ← shutdown 상태
Serial1/1         unassigned      YES unset   down                  down  ← 상대방 shutdown
Serial1/2         unassigned      YES unset   up                    down  ← Encapsulation 불일치/Clock 없음
Serial1/3         192.168.12.1    YES manual  up                    up    ← 정상

R# show interface fastethernet 0/0
R# show arp
R# show ip route
```

---

## UTP Cable 종류

| 종류 | 용도 | 연결 예 |
|------|------|---------|
| **Straight-Through** | 다른 장비 간 연결 | PC-Switch, Switch-Router |
| **Crossover** | 같은 장비 간 연결 | PC-PC, Switch-Switch, Router-Router, PC-Router |
| **Roll-over (Console)** | Console 접속 | PC-Router Con 포트, PC-Switch Con 포트 |

---

## Password 설정

### Console Password

PC의 Serial Cable로 Router Console Port에 직접 접속 시 적용

```
Router(config)# line console 0
Router(config-line)# password WORD
Router(config-line)# login
```

### VTY Password

Telnet(TCP 23) / SSH(TCP 22)를 통한 원격 접속 시 적용

- Router: 0-4 (5개 포트), Switch: 0-15 (16개 포트)

```
Router(config)# line vty 0 4
Router(config-line)# password WORD
Router(config-line)# login
```

### AUX Password

Modem을 통한 원격 접속 시 적용

```
Router(config)# line aux 0
Router(config-line)# password WORD
Router(config-line)# login
```

### Enable Secret / Enable Password

Console/VTY/AUX 접속 후 Privilege Mode 전환 시 적용

```
Router(config)# enable secret WORD    ← MD5 Hash 암호화 저장 (권장)
Router(config)# enable password WORD  ← 평문 저장
```

> Enable Secret와 Enable Password가 동시에 설정된 경우 **Enable Secret가 우선** 적용

---

## CDP (Cisco Discovery Protocol)

직접 연결된 장비 정보를 자동으로 수집하는 Cisco 전용 Layer 2 프로토콜

- 60초마다 CDP 메시지 전송, 180초 후 만료
- Cisco 장비 간에만 동작

### 수집 정보

| 항목 | 내용 |
|------|------|
| Device ID | 인접 장비 Hostname |
| Local Interface | 자신의 인터페이스 |
| Port ID | 인접 장비 인터페이스 |
| Platform | 인접 장비 모델명 |
| Capability | Router / Switch |
| Next-hop IP | 인접 인터페이스 IP |

### 명령어

```
R# show cdp
R# show cdp neighbor
R# show cdp neighbor detail

R(config)# no cdp run              ← CDP 전체 비활성화
R(config)# cdp timer <sec>         ← 전송 주기 변경
R(config)# cdp holdtime <sec>      ← 유지 시간 변경
```

---

## IOS XE 16.x / 17.x 관점

### NETCONF / RESTCONF (`IOS XE 16.x+`)

```
! IOS XE 16.x+ — NETCONF 활성화
R(config)# netconf-yang
R(config)# restconf

R# show platform software status control-processor brief
R# show version
```

### Interface 확인 강화

```
! IOS XE 16.x+ — 인터페이스 세부 통계
R# show interfaces GigabitEthernet0/0/0 counters
R# show ip interface GigabitEthernet0/0/0
```
