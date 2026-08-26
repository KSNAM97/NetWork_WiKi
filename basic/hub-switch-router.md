# Hub · Switch · Router

## 네트워크란

- **사회적 관점**: 공통 목적을 가진 그룹 (클럽, 동호회, Cafe 등)
- **IT 관점**: 효율적인 통신을 위해 장비들을 연결한 구조 (사무실, 빌딩, PC방 등)
- 네트워크가 확대될수록 효율적인 데이터 전송을 위한 **Protocol(규약)**이 필요

### Encapsulation (캡슐화)

데이터만으로는 목적지를 알 수 없으므로, 데이터에 **Source/Destination Address를 추가**하는 과정

---

## LAN vs WAN

| 구분 | LAN (Local Area Network) | WAN (Wide Area Network) |
|------|--------------------------|-------------------------|
| 범위 | 자신이 속한 동일 네트워크 | 자신이 속하지 않은 외부 네트워크 |
| 대표 장비 | NIC(랜카드), Switch, TP Cable | Router |
| 대표 Protocol | **Ethernet** (IEEE 표준) | **IP** (IANA 관리) |
| 주소 체계 | MAC Address (16진수 48bit) | IP Address (10진수 32bit) |

### Ethernet 속도

| 규격 | 속도 |
|------|------|
| Ethernet | 10 Mbps |
| FastEthernet | 100 Mbps |
| GigabitEthernet | 1,000 Mbps |

### MAC Address 구조

```
예: 00-E0-4C-B0-C4-81
    └──┬──┘ └──┬──┘
      OUI     일련번호
  (IEEE 임대)  (제조사 부여)
```

---

## Layer별 데이터 단위 및 주소

| Layer | 대표 장비 | 주소 | 데이터 단위 | 역할 |
|-------|----------|------|------------|------|
| L4 (Transport) | - | Port number (16bit) | **Segment** | 서비스 식별 |
| L3 (Network) | Router | IP Address (32bit) | **Packet** | 원격 네트워크 통신 |
| L2 (Data Link) | Switch | MAC Address (48bit) | **Frame** | 동일 네트워크 통신 |

### 주요 L4 서비스 포트

| Protocol | 서비스 | Port |
|----------|--------|------|
| TCP | HTTP | 80 |
| TCP | FTP (데이터/제어) | 20 / 21 |
| TCP | Telnet | 23 |
| TCP | SSH | 22 |
| UDP | DNS | 53 |
| UDP | DHCP (서버/클라이언트) | 67 / 68 |
| UDP | TFTP | 69 |

---

## Hub

- **Layer 1** 대표 장비
- 동일 네트워크 내 장비 간 통신에 사용
- 전기적인 신호를 증폭하여 데이터를 Forwarding
- **Collision Domain을 공유**하는 장비

## Switch

- **Layer 2** 대표 장비
- 동일 네트워크 내 장비 간 통신에 사용
- Ethernet Header에 포함된 **MAC Address**(16진수, 48bit)를 사용해 통신
- 프레임 수신 시 Destination MAC을 MAC Address Table에서 조회해 Forwarding
- **Collision Domain을 분할**, Broadcast Domain을 공유
- Cisco IOS에 의존하는 **하드웨어 기반** 장비

## Router

- **Layer 3** 대표 장비
- 다른 네트워크로의 통신에 사용
- IP Header에 포함된 **IP Address**(10진수, 32bit)를 사용해 데이터 전송
- 프레임 수신 시 Destination IP를 Routing Table에서 조회해 Forwarding
- **Broadcast Domain을 분할**하는 장비
- Cisco IOS에 의존하는 **소프트웨어 기반** 장비

---

## 장비 비교

| 구분 | Layer | 주소 | Collision Domain | Broadcast Domain |
|------|-------|------|-----------------|-----------------|
| Hub | L1 | - | 공유 | 공유 |
| Switch | L2 | MAC | 분할 | 공유 |
| Router | L3 | IP | 분할 | 분할 |

---

## IOS XE 16.x / 17.x 관점

### Switch — ASIC 기반 하드웨어 포워딩 (`IOS XE 16.x+`)

IOS XE 16.x부터 Catalyst 시리즈는 Cisco IOS와 Linux 커널을 결합한 아키텍처로 전환

| 항목 | IOS 15.x (Phase 2) | IOS XE 16.x+ |
|------|-------------------|--------------|
| 기반 | 단일 IOS 이미지 | IOS XE (Linux + IOS 프로세스) |
| 관리 인터페이스 | Console/VTY | Console/VTY + NETCONF/RESTCONF |
| 자동화 지원 | 없음 | Python, gRPC, YANG 모델 |
| 대표 플랫폼 | Catalyst 2960/3560 | Catalyst 9000 시리즈 |

### Switch 진단 명령어 변화 (`IOS XE 16.x+`)

```
! MAC Address Table 확인 (IOS XE 동일 문법)
SW# show mac address-table
SW# show mac address-table dynamic
SW# show mac address-table count

! IOS XE 16.x+ 추가 — 인터페이스별 필터
SW# show mac address-table interface GigabitEthernet1/0/1
SW# show mac address-table vlan 10
```

### Router — Cisco IOS XE 17.x

```
! IOS XE 17.x — 플랫폼 정보 확인
R# show platform software status control-processor brief
R# show version   ! IOS XE 버전 및 플랫폼 확인
```

> IOS XE 16.x 이상에서 Router/Switch 경계가 희미해짐 — Catalyst 9000은 L2/L3 모두 처리하며 SD-Access Fabric의 Edge/Border 노드로 동작
