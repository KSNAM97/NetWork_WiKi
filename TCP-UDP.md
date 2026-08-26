# TCP · UDP

## TCP (Transmission Control Protocol)

- **연결 지향** 프로토콜 — 통신 전 3-way Handshake로 연결 수립
- **신뢰성** 보장 — 재전송, 순서 제어, 흐름 제어
- Protocol 번호: **6**
- 주요 사용: HTTP(80), HTTPS(443), SSH(22), Telnet(23), FTP(21)

### 3-way Handshake

```
Client                          Server
  │──── SYN (seq=0) ──────────────►│
  │◄─── SYN-ACK (seq=0, ack=1) ───│
  │──── ACK (seq=1, ack=1) ───────►│
  │           연결 수립             │
```

| 단계 | 방향 | 플래그 | seq | ack |
|------|------|--------|-----|-----|
| 1 | Client → Server | SYN | 0 | - |
| 2 | Server → Client | SYN, ACK | 0 | 1 |
| 3 | Client → Server | ACK | 1 | 1 |

---

## UDP (User Datagram Protocol)

- **비연결 지향** 프로토콜 — 연결 수립 없이 데이터 전송
- **신뢰성 없음** — 재전송 없음, 속도 우선
- Protocol 번호: **17**
- 주요 사용: DNS(53), DHCP(67/68), SNMP(161), NTP(123)

---

## 헤더 비교

| 항목 | TCP | UDP |
|------|-----|-----|
| 연결 | 연결 지향 | 비연결 지향 |
| 신뢰성 | 높음 (재전송) | 없음 |
| 속도 | 상대적으로 느림 | 빠름 |
| 헤더 크기 | 20byte (최소) | 8byte |
| 오버헤드 | 높음 | 낮음 |

---

## 패킷 흐름 예시

```
PC (10.1.62.112) ──→ DNS Server (168.126.63.1)

Ethernet  | SA: PC_MAC        DA: GW_MAC
IP        | SA: 10.1.62.112   DA: 168.126.63.1
UDP       | SA: 64489         DA: 53 (DNS)
DNS       | Query: www.naver.com

DNS Server ──→ PC

Ethernet  | SA: GW_MAC        DA: PC_MAC
IP        | SA: 168.126.63.1  DA: 10.1.62.112
UDP       | SA: 53            DA: 64489
DNS       | Response: 223.130.200.236
```

---

## IOS XE 16.x / 17.x 관점

### TCP 세션 추적 — NBAR2 (`IOS XE 16.x+`)

IOS XE 16.x부터 NBAR2(Network Based Application Recognition)가 기본 탑재 — 포트 번호 외 **페이로드 패턴**으로 애플리케이션 식별

```
! 인터페이스별 프로토콜 트래픽 통계
R# show ip nbar protocol-discovery
R# show ip nbar protocol-discovery interface GigabitEthernet0/0 stats byte-count top-n 10

! TCP 연결 상태 확인 (IOS XE 동일)
R# show tcp brief       ! TCP 세션 목록
R# show tcp statistics  ! 재전송·오류 통계
```

### QUIC (UDP 443) 인식 (`IOS XE 17.x`)

HTTP/3는 TCP 대신 **UDP 443**을 사용하는 QUIC 기반 — 기존 ACL에서 UDP 443 차단 시 Google 서비스 등에 영향

```
! IOS XE 17.x — NBAR2로 QUIC 분류
R# show ip nbar protocol-discovery | include quic

! QUIC 차단 예시 (ACL 적용)
ip access-list extended BLOCK_QUIC
 deny udp any any eq 443   ! QUIC 차단 → 클라이언트가 TCP 443(TLS 1.3)으로 폴백
 permit ip any any
```

| 버전 | TCP/UDP 관련 변화 |
|------|----------------|
| IOS 15.x (Phase 2) | 포트 번호 기반 분류, 기본 ACL |
| IOS XE 16.x+ | NBAR2 페이로드 분류, DTLS 지원 |
| IOS XE 17.x | QUIC(UDP 443) NBAR2 인식, TLS 1.3 기본 지원 |
