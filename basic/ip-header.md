# IP Header

Layer 3 Protocol. Local → Remote 네트워크 통신 시 사용.

---

## IP Header 구조

| 필드 | 크기 | 설명 |
|------|------|------|
| Version | 4bit | IP 버전 (IPv4 / IPv6) |
| Header Length | 4bit | IP Header 크기 (가변형) |
| Type-Of-Service | 8bit | QoS 우선순위 (IP Precedence, DSCP, ECN) |
| Total Length | 16bit | IP Header + Data 합산 크기 |
| Identification | 16bit | 단편화된 조각을 구분하는 식별자 |
| Flag | 3bit | 단편화 제어 |
| Fragmentation Offset | 13bit | 단편화 조각의 원본 위치 |
| TTL | 8bit | Loop 방지용 Hop 수 제한 |
| Protocol | 8bit | 상위 Protocol 식별 |
| Header Checksum | 16bit | IP Header 오류 검출 (재전송 X) |
| Source IP Address | 32bit | 출발지 IP 주소 |
| Destination IP Address | 32bit | 목적지 IP 주소 |

---

## 주요 필드 상세

### Type-Of-Service (ToS)

- **IP Precedence**: 앞 3bit — 8단계 우선순위
- **DSCP**: 6bit — 64단계 우선순위 (더 세분화된 QoS)
- **ECN**: 마지막 2bit — 버퍼 부족/처리 불가 시 사용

### Flag (3bit)

```
0  DF  MF
│   │   └─ More Fragment: 1=뒤에 단편화 데이터 있음, 0=마지막 조각
│   └───── Don't Fragment: 1=단편화 금지, 0=단편화 허용
└───────── 예약 (항상 0)
```

### Protocol 번호

| Protocol | 번호 |
|----------|------|
| ICMP | 1 |
| TCP | 6 |
| UDP | 17 |
| EIGRP | 88 |
| OSPF | 89 |

---

## MTU와 단편화

- **MTU (Maximum Transmission Unit)**: 장비가 한 번에 전송 가능한 최대 크기 (기본 **1500 Byte**)
- 전송할 데이터가 MTU를 초과하면 → **단편화(Fragmentation)** 실시
- 수신 측은 `Identification` + `Fragmentation Offset`으로 조각을 원본으로 재조립

---

## ARP (Address Resolution Protocol)

- **Layer 3 주소(IP)**를 사용하여 **Layer 2 주소(MAC)**를 학습하는 Protocol
- 상대방 IP는 알지만 MAC Address를 모를 때 사용

```
Host# show arp
```

---

## 통신 흐름 예시 (PC → Web Server)

```
PC → DNS Server (UDP/53)
  SA MAC: PC    DA MAC: GW
  SA IP: PC     DA IP: DNS Server

DNS Server → PC (응답)
  SA IP: DNS    DA IP: PC
  DNS 응답: www.naver.com = 223.130.200.236

PC → Web Server (TCP 3-way Handshake)
  SYN  → SA Port: 57787  DA Port: 443  seq=0
  SYN-ACK ← seq=0  ack=1
  ACK  → seq=1  ack=1
```

---

## IOS XE 16.x / 17.x 관점

> IPv4 Header 구조 자체(RFC 791)는 변경 없음 — IP Header 필드와 크기는 IOS 버전에 무관하게 동일

### IPv6 기본 헤더 비교 (참고)

| 항목 | IPv4 | IPv6 |
|------|------|------|
| 헤더 크기 | 20byte (가변) | 40byte (고정) |
| 주소 길이 | 32bit | 128bit |
| Fragmentation | 라우터/호스트 모두 가능 | **호스트만** 가능 (DF 항상 ON) |
| Header Checksum | 있음 | **없음** (상위 계층에서 처리) |
| TTL 필드 명칭 | TTL | Hop Limit |
| Protocol 필드 명칭 | Protocol | Next Header |

### IOS XE에서 MTU/단편화 확인

```
! 인터페이스 MTU 확인 및 변경
interface GigabitEthernet0/1
 ip mtu 1500          ! IP MTU (기본 1500)

! Path MTU Discovery 확인
Router# show ip interface GigabitEthernet0/1
! → IP MTU: 1500 bytes 항목으로 확인

! DF bit 강제 설정 (터널 인터페이스 등)
interface Tunnel0
 ip tcp adjust-mss 1452    ! TCP MSS 조정으로 단편화 방지
```

### DSCP / QoS 마킹 (IOS XE 17.x)

ToS 필드의 DSCP 값은 IOS XE에서 MQC(Modular QoS CLI)를 통해 마킹

```
! Class-map으로 트래픽 분류
class-map match-any VOICE
 match dscp ef                ! EF = Expedited Forwarding (DSCP 46)

! Policy-map으로 DSCP 마킹
policy-map MARK_TRAFFIC
 class VOICE
  set dscp ef

! 인터페이스에 적용
interface GigabitEthernet0/1
 service-policy input MARK_TRAFFIC
```

| DSCP 값 | 이름 | 용도 |
|---------|------|------|
| 46 (101110) | EF (Expedited Forwarding) | VoIP, 실시간 음성 |
| 34 (100010) | AF41 | 영상 회의 |
| 0 | BE (Best Effort) | 일반 데이터 |
