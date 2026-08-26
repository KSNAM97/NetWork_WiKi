# VPN (Virtual Private Network)

공용 네트워크(인터넷)를 통해 **암호화된 전용 통신 채널**을 구성하는 기술

---

## GRE Tunnel

- **Generic Routing Encapsulation**
- IP 패킷을 캡슐화하여 논리적 터널 구성
- **암호화 없음** — 단순 캡슐화
- Multicast, 브로드캐스트 지원 → 라우팅 프로토콜 전달 가능

```
R1(config)# interface tunnel 0
R1(config-if)# ip address 172.16.0.1 255.255.255.252
R1(config-if)# tunnel source fastethernet 0/0
R1(config-if)# tunnel destination [R2 공인 IP]
R1(config-if)# tunnel mode gre ip
```

---

## IPsec

- 데이터 **암호화 + 인증** 제공
- GRE보다 보안성 높음
- Multicast 미지원 (라우팅 프로토콜 직접 불가)

### 주요 구성 요소

| 구성 요소 | 설명 |
|-----------|------|
| ISAKMP/IKE | 키 교환 및 SA(Security Association) 협상 |
| ESP | 암호화 + 인증 |
| AH | 인증만 (암호화 없음) |

### 설정 예시

```
# ISAKMP Policy (Phase 1)
R1(config)# crypto isakmp policy 10
R1(config-isakmp)# encryption aes
R1(config-isakmp)# hash sha
R1(config-isakmp)# authentication pre-share
R1(config-isakmp)# group 2
!
R1(config)# crypto isakmp key [비밀키] address [Peer IP]

# Transform Set (Phase 2)
R1(config)# crypto ipsec transform-set TSET esp-aes esp-sha-hmac

# Crypto Map
R1(config)# crypto map CMAP 10 ipsec-isakmp
R1(config-crypto-map)# set peer [Peer IP]
R1(config-crypto-map)# set transform-set TSET
R1(config-crypto-map)# match address 101
!
R1(config)# interface fastethernet 0/0
R1(config-if)# crypto map CMAP
```

---

## GRE over IPsec

- GRE 터널에 IPsec 암호화 적용
- Multicast 지원 (GRE) + 암호화 (IPsec)
- 라우팅 프로토콜 실행 가능하면서 보안 확보

```
구성 순서:
1. GRE Tunnel Interface 구성
2. IPsec 설정 (ISAKMP + Transform Set + Crypto Map)
3. Crypto Map을 물리 Interface에 적용
4. Tunnel Interface에서 라우팅 프로토콜 실행
```

---

## DMVPN (Dynamic Multipoint VPN)

- Hub-and-Spoke 구조에서 **Spoke 간 직접 통신** 가능
- 동적으로 터널을 생성하여 확장성 우수

### 주요 구성 요소

| 구성 요소 | 설명 |
|-----------|------|
| **mGRE** | Multipoint GRE — 하나의 터널 인터페이스로 다중 피어 지원 |
| **NHRP** | Next Hop Resolution Protocol — IP ↔ NBMA 주소 매핑 |
| **IPsec** | 암호화 (선택적) |

### Hub 설정

```
R-HUB(config)# interface tunnel 0
R-HUB(config-if)# ip address 10.0.0.1 255.255.255.0
R-HUB(config-if)# tunnel source fastethernet 0/0
R-HUB(config-if)# tunnel mode gre multipoint
R-HUB(config-if)# ip nhrp network-id 1
R-HUB(config-if)# ip nhrp map multicast dynamic
```

### Spoke 설정

```
R-SPOKE(config)# interface tunnel 0
R-SPOKE(config-if)# ip address 10.0.0.2 255.255.255.0
R-SPOKE(config-if)# tunnel source fastethernet 0/0
R-SPOKE(config-if)# tunnel mode gre multipoint
R-SPOKE(config-if)# ip nhrp network-id 1
R-SPOKE(config-if)# ip nhrp nhs [Hub 터널 IP]
R-SPOKE(config-if)# ip nhrp map [Hub 터널 IP] [Hub NBMA IP]
R-SPOKE(config-if)# ip nhrp map multicast [Hub NBMA IP]
```

---

## VPN 종류 비교

| 종류 | 암호화 | Multicast | 확장성 | 특징 |
|------|--------|-----------|--------|------|
| GRE | X | O | 보통 | 단순 캡슐화 |
| IPsec | O | X | 보통 | 높은 보안 |
| GRE+IPsec | O | O | 보통 | 보안 + 라우팅 |
| DMVPN | O | O | 우수 | 동적 Spoke 간 통신 |

---

## 정보 확인

```
R# show crypto isakmp sa
R# show crypto ipsec sa
R# show interface tunnel 0
R# show ip nhrp
```

---

## CBAC (Context-Based Access Control)

IOS 12.x~15.x 시대의 **Stateful Firewall** 기능 — Layer 7까지 검사하여 동적 ACL 구멍 생성

- `ip inspect`: 세션 State Table 관리
- TCP 세션: 3600초, UDP: 30초, ICMP: 10초 타임아웃
- Half-open 연결 임계값으로 DoS 방어
- Audit-trail 로깅 지원

```
! CBAC 검사 규칙 정의
ip inspect name CBAC tcp  audit-trail on   ! TCP 세션 추적 + 로그
ip inspect name CBAC udp
ip inspect name CBAC icmp
!
! 인터페이스 적용 — 내부에서 외부 방향
interface FastEthernet 0/0      ! 외부(인터넷) 인터페이스
 ip access-group ACL in         ! 인바운드 ACL로 외부 유입 차단
 ip inspect CBAC out            ! 아웃바운드 세션 추적 → 돌아오는 트래픽 자동 허용
```

```
! DoS 방어 임계값 설정
ip inspect max-incomplete high 500   ! Half-open 세션 최대 500개
ip inspect max-incomplete low  400   ! 해소 임계값
```

```
! 확인
R# show ip inspect session      ! 현재 활성 세션 State Table
R# show ip inspect config       ! CBAC 규칙 목록
R# show ip inspect all          ! 세션 + 규칙 + 통계 통합 확인
```

> ⚠️ IOS XE 16.x 이상에서는 CBAC 대신 **Zone-Based Policy Firewall(ZBF)** 사용 권장

---

## IPsec 이중화 (HSRP 연동)

IPsec VPN의 Active Router를 HSRP로 이중화 — HSRP Virtual IP를 IPsec Peer로 사용하여 Failover 시에도 원격 피어 설정 변경 불필요

```
! ISAKMP Keepalive — Dead Peer 감지 (DPD)
crypto isakmp keepalive 10      ! 10초 간격으로 피어 생존 확인
!
! HSRP 그룹에 이름 부여 (crypto map 연동용)
interface fastethernet 0/1
 standby 2 ip [Virtual IP]
 standby 2 name HSRP_IPSEC      ! HSRP 그룹 이름 — crypto map과 연결
!
! crypto map에 HSRP 이중화 연동
interface fastethernet 0/1
 no crypto map IPSEC_REDUNDANCY         ! 기존 단순 crypto map 제거
 crypto map IPSEC_REDUNDANCY redundancy HSRP_IPSEC   ! HSRP 연동으로 재적용
```

동작 원리:
- **Active Router**: crypto map 활성화 → VPN 세션 처리
- **Standby Router**: crypto map 비활성화 상태 대기
- HSRP Failover 시: Standby가 Active로 전환 → crypto map 자동 활성화
- 원격 피어는 Virtual IP로 연결하므로 설정 변경 없음

```
! 확인
R# show crypto isakmp peers              ! 현재 ISAKMP 피어 상태
R# show crypto engine connections active ! 암호화 엔진 활성 세션
R# show standby brief                    ! HSRP Active/Standby 상태
```

---

## IOS XE 16.x / 17.x 최신 트렌드

### IKEv2 (`IOS XE 15.2+ / 16.x 표준화`)

Phase 2 자료의 IPsec은 **IKEv1** 기반. IOS XE 16.x 이상에서는 **IKEv2**가 표준.

| 항목 | IKEv1 (IOS 15.x) | IKEv2 (IOS XE 16.x+) |
|------|------------------|----------------------|
| 교환 횟수 | Phase 1: 6패킷, Phase 2: 3패킷 | **4패킷** (더 빠름) |
| 비대칭 인증 | 불가 | **가능** (한쪽 PSK, 다른쪽 인증서) |
| MOBIKE | 미지원 | **지원** (IP 변경 시 재협상 없음) |
| DoS 방어 | 취약 | **Cookie 교환** 포함 |

```
! IKEv2 설정 (IOS XE 16.x+)
! Phase 1 — IKEv2 Proposal
crypto ikev2 proposal IKEv2_PROP
 encryption aes-cbc-256
 integrity sha512
 group 20              ! ECDH 384-bit (IOS 15.x는 group 5/14까지)

crypto ikev2 policy IKEv2_POL
 proposal IKEv2_PROP

! IKEv2 Profile (PSK 인증)
crypto ikev2 keyring IKEv2_KEYS
 peer PEER_R2
  address 2.2.2.2
  pre-shared-key SECRET_KEY

crypto ikev2 profile IKEv2_PROF
 match identity remote address 2.2.2.2
 authentication remote pre-share
 authentication local pre-share
 keyring local IKEv2_KEYS

! Phase 2 — Transform Set (동일)
crypto ipsec transform-set TSET esp-aes 256 esp-sha512-hmac
 mode tunnel

! Crypto Map
crypto map CMAP 10 ipsec-isakmp
 set peer 2.2.2.2
 set ikev2-profile IKEv2_PROF
 set transform-set TSET
 match address 101
```

### FlexVPN (`IOS XE 16.x — DMVPN 차세대`)

DMVPN의 후속 — IKEv2 기반으로 통합, 설정 간소화

```
! FlexVPN Hub 설정 (IOS XE 16.x+)
crypto ikev2 authorization policy FLEX_POLICY
 route set interface            ! Spoke에 분배할 경로 자동 설정

interface Tunnel0
 ip address 10.0.0.1 255.255.255.0
 tunnel source GigabitEthernet0/1
 tunnel mode ipsec ipv4         ! FlexVPN은 IPsec 직접 터널
 tunnel protection ipsec profile IPSEC_PROF
```

### AES-GCM 암호화 (`IOS XE 16.x+`)

Phase 2 자료의 `esp-aes esp-sha-hmac` 대신 **AES-GCM** 사용 권장 (인증+암호화 통합)

```
! IOS XE 16.x+ — AES-256-GCM Transform Set
crypto ipsec transform-set MODERN_TSET esp-aes-gcm 256
 mode tunnel
!  → AES-GCM은 MAC과 암호화를 한 번에 처리 (esp-sha-hmac 불필요)
```

```
! 확인 명령어 (IOS XE 16.x+)
R# show crypto ikev2 sa          ! IKEv2 SA 상태
R# show crypto ikev2 session     ! 세션 상세 (IKEv1과 다른 명령)
R# show crypto ipsec sa detail   ! ESP 패킷 암호화 카운터
```

> IOS 15.x Phase 2 자료의 `crypto isakmp` 명령어는 IKEv1 전용.  
> IOS XE 16.x 이상에서 IKEv2 사용 시 `crypto ikev2` 명령어로 전환.
