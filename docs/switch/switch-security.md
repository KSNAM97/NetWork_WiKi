# Switch 보안

---

## Port Security

특정 포트에 **허용되는 MAC Address를 제한**하여 무단 접속 방지

### 설정 순서

```
SW(config)# interface fastethernet 0/1
SW(config-if)# switchport mode access
SW(config-if)# switchport port-security maximum 1          ← 최대 MAC 수 (기본 1)
SW(config-if)# switchport port-security mac-address [MAC]  ← 정적 MAC 등록
SW(config-if)# switchport port-security violation shutdown  ← 위반 시 동작
SW(config-if)# switchport port-security                    ← 마지막에 활성화
```

### Violation 모드

| 모드 | 동작 | 카운터 증가 | 포트 상태 |
|------|------|------------|-----------|
| **protect** | 위반 트래픽만 차단 | X | 유지 |
| **restrict** | 위반 트래픽 차단 + 로그 | O | 유지 |
| **shutdown** | 포트 err-disable 전환 | O | err-disable |

### 실습 예시

```
SW1(config)# interface fastethernet 0/1
SW1(config-if)# switchport mode access
SW1(config-if)# switchport port-security maximum 1
SW1(config-if)# switchport port-security mac-address 00D0.BC6D.42E3
SW1(config-if)# switchport port-security violation shutdown
SW1(config-if)# switchport port-security
```

### 확인

```
SW# show port-security
Secure Port  MaxSecureAddr  CurrentAddr  SecurityViolation  Security Action
             (Count)        (Count)      (Count)
-----------------------------------------------------------------------
      Fa0/1       1              1               0           Shutdown
      Fa0/2       1              1               0           Shutdown

SW# show port-security interface fastEthernet 0/1
Port Security       : Enabled
Port Status         : Secure-up
Violation Mode      : Shutdown
Maximum MAC Addresses: 1
Total MAC Addresses : 1
Configured MAC Addresses: 1
Last Source Address : 00D0.BC6D.42E3:1
Security Violation Count: 0
```

### 실습 예시: Violation 모드를 shutdown에서 restrict로 변경

> 조건: fa0/1, fa0/2, fa0/3, fa0/20에 이미 shutdown 모드로 Port Security가 설정되어 있다. 이제 위반 시 포트를 내려버리지 않고 로그만 남기도록 정책을 바꿔야 한다.

```
SW1(config)# interface fastethernet 0/1
SW1(config-if)# switchport port-security violation restrict
SW1(config-if)# switchport port-security
!
SW1(config)# interface fastethernet 0/2
SW1(config-if)# switchport port-security violation restrict
SW1(config-if)# switchport port-security
```

MAC 주소나 maximum 값은 그대로 두고 `violation` 모드만 restrict로 바꾸면, 이후 등록되지 않은 MAC이 들어와도 포트는 살아있고 `show port-security`의 SecurityViolation 카운터만 증가한다.

### err-disable 해제

```
SW(config-if)# shutdown
SW(config-if)# no shutdown
```

자동 복구 설정:
```
SW(config)# errdisable recovery cause psecure-violation
SW(config)# errdisable recovery interval 300
```

---

## Port Security Sticky

관리자가 MAC을 직접 입력하지 않아도 **처음 연결된 장비의 MAC을 자동으로 학습 후 고정**

```
SW(config)# interface fastethernet 0/1
SW(config-if)# switchport mode access
SW(config-if)# switchport port-security maximum 1
SW(config-if)# switchport port-security mac-address sticky   ← 자동 학습 후 저장
SW(config-if)# switchport port-security violation restrict
SW(config-if)# switchport port-security
```

> `show port-security address` — Sticky로 학습된 MAC 확인 가능

---

## Storm Control

포트로 유입되는 **Broadcast/Multicast/Unicast 트래픽의 임계치**를 설정하여 네트워크 폭풍 방지

### CPU 소모율 기반

```
SW(config-if)# storm-control broadcast level 30        ← 30% 초과 시 차단
SW(config-if)# storm-control broadcast level 30 20    ← 30% 초과 시 차단, 20% 이하 시 복구
```

### PPS(초당 패킷 수) 기반

```
SW(config-if)# storm-control broadcast level pps 300 100
```
← 초당 300개 초과 시 차단, 100개 이하 시 복구

### 트래픽 초과 시 포트 err-disable

```
SW(config-if)# storm-control broadcast level 30
SW(config-if)# storm-control action shutdown
```

### 실습 예시: Unicast 트래픽 기준 err-disable + 자동 복구

> 조건: SW2의 fa0/23으로 들어오는 Unicast 트래픽 때문에 CPU 소모율이 1% 이상 올라가면 해당 포트를 err-disable로 전환하고, 1분 뒤 자동으로 복구되어야 한다.

```
SW2(config)# interface fastethernet 0/23
SW2(config-if)# storm-control unicast level 1
SW2(config-if)# storm-control action shutdown
SW2(config-if)# exit
!
SW2(config)# errdisable recovery cause storm-control
SW2(config)# errdisable recovery interval 60
```

`ping ... repeat 1000 size 1000` 처럼 대량의 Unicast 트래픽을 흘리면 fa0/23이 즉시 err-disable로 전환되는 것을 `show storm-control unicast`와 `show interface fa0/23 status`로 확인할 수 있고, 60초 후 별도 조작 없이 포트가 다시 살아난다.

### err-disable 자동 복구

```
SW(config)# errdisable recovery cause storm-control
SW(config)# errdisable recovery interval 60
```

### 확인

```
SW# show storm-control broadcast
Interface   Filter State  Upper     Lower    Current
-----------+-------------+---------+---------+-------
Fa0/1       Forwarding    30.00%    20.00%   0.00%
```

---

## SPAN (Switched Port Analyzer)

특정 포트의 트래픽을 **모니터링 장비로 복사**하는 기능 (Mirroring)

### Local SPAN

모니터링 장비와 감시 대상이 **같은 스위치**에 연결된 환경

```
SW(config)# monitor session 1 source interface fastethernet 0/1 both
SW(config)# monitor session 1 destination interface fastethernet 0/20
```

| 방향 옵션 | 설명 |
|-----------|------|
| `both` | 송수신 모두 복사 |
| `tx` | 송신만 복사 |
| `rx` | 수신만 복사 |

### 확인

```
SW# show monitor session 1
Session 1
----------
Type            : Local Session
Source Ports    :
    Both        : Fa0/1
Destination Ports: Fa0/20
```

### 실습 예시: PC 트래픽을 Wireshark로 확인

> 조건: Fa0/1에 연결된 PC(192.168.1.100)의 송수신 트래픽을 Fa0/20에 연결된 Monitoring PC에서 Wireshark로 그대로 확인해야 한다.

```
SW1(config)# monitor session 1 source interface fastethernet 0/1 both
SW1(config)# monitor session 1 destination interface fastethernet 0/20
```

이 상태에서 PC에서 `ping`이나 `telnet`을 실행하면, Fa0/1과는 직접 연결되어 있지 않은 Fa0/20의 Monitoring PC에서도 Wireshark로 그 트래픽이 그대로 캡처된다 — SPAN 목적지 포트는 정상적인 통신 포트가 아니라 오직 복사된 트래픽을 받는 용도로만 쓰인다.

---

### Remote SPAN (RSPAN)

모니터링 장비와 감시 대상이 **서로 다른 스위치**에 연결된 환경

```
                    Fa0/24                Fa0/24
       SW1 ────────────────────────── SW2
        │                                │
       Fa0/1                           Fa0/20
        │                                │
      Server                         Monitoring
```

**SW1 (Source Switch)**

```
SW1(config)# vlan 999
SW1(config-vlan)# remote-span
!
SW1(config)# monitor session 1 source interface fastethernet 0/1 both
SW1(config)# monitor session 1 destination remote vlan 999 reflector-port fastethernet 0/22
```

> `reflector-port`: 복사 트래픽을 처리할 미사용 포트 — 해당 포트의 CPU를 사용

**SW2 (Destination Switch)**

```
SW2(config)# vlan 999
SW2(config-vlan)# remote-span
!
SW2(config)# monitor session 1 source remote vlan 999
SW2(config)# monitor session 1 destination interface fastethernet 0/20
```

### 확인

```
SW1# show monitor session 1
Type            : Remote Source Session
Source Ports    :
    Both        : Fa0/1
Dest RSPAN VLAN : 999

SW2# show monitor session 1
Type                 : Remote Destination Session
Source RSPAN VLAN    : 999
Destination Ports    : Fa0/20
```

---

## 정보 확인 요약

```
SW# show port-security
SW# show port-security address
SW# show port-security interface fa0/1
SW# show storm-control broadcast
SW# show storm-control unicast
SW# show monitor session 1
```

---

## IOS XE 16.x / 17.x 최신 트렌드

### DHCP Snooping (`IOS XE 16.x+`)

가짜 DHCP 서버를 차단 — Untrusted 포트에서 오는 DHCP Offer/ACK 패킷 차단

```
! DHCP Snooping 전역 활성화 (IOS XE 16.x+)
SW(config)# ip dhcp snooping
SW(config)# ip dhcp snooping vlan 10,20     ! 대상 VLAN 지정

! Uplink 포트(진짜 DHCP 서버 방향)는 Trusted
SW(config)# interface GigabitEthernet1/0/1
SW(config-if)# ip dhcp snooping trust       ! DHCP 서버 or Uplink 포트

! 사용자 포트는 Untrusted (기본값) — Offer/ACK 차단
! Rate Limit — DHCP Discovery 공격 방어
SW(config-if)# ip dhcp snooping limit rate 15   ! 초당 15패킷 초과 시 err-disable
```

### Dynamic ARP Inspection (DAI) (`IOS XE 16.x+`)

ARP Spoofing 방어 — DHCP Snooping 바인딩 테이블과 연동하여 위조 ARP 차단

```
! DAI 활성화 (DHCP Snooping 선행 필요)
SW(config)# ip arp inspection vlan 10,20

! Uplink 포트는 Trusted
SW(config)# interface GigabitEthernet1/0/1
SW(config-if)# ip arp inspection trust

! 확인
SW# show ip arp inspection vlan 10
SW# show ip arp inspection statistics
```

### IP Source Guard (`IOS XE 16.x+`)

DHCP Snooping 바인딩 기반 — 허가된 IP/MAC 조합 외 패킷 차단

```
! IP Source Guard 활성화 (DHCP Snooping 선행 필요)
SW(config)# interface GigabitEthernet1/0/5
SW(config-if)# ip verify source            ! IP만 검증
SW(config-if)# ip verify source port-security  ! IP + MAC 함께 검증

! 확인
SW# show ip source binding
SW# show ip verify source
```

### 802.1X + MACsec (`IOS XE 17.x+`)

유선 포트 레벨 암호화 — 802.1X 인증 후 MACsec(IEEE 802.1AE)으로 L2 암호화

```
! MACsec 설정 (IOS XE 17.x — 하드웨어 지원 필요)
! Key Agreement — MKA (MACsec Key Agreement)
mka policy MKA_POLICY
 macsec-cipher-suite gcm-aes-256    ! AES-256-GCM 암호화

interface GigabitEthernet1/0/1
 authentication port-control auto
 mka policy MKA_POLICY
 macsec                             ! MACsec 활성화

SW# show macsec summary             ! MACsec 상태 확인
SW# show mka sessions               ! MKA 세션 확인
```

> IOS 15.x Phase 2 자료는 Port Security(MAC 제한), Storm Control, SPAN 위주.  
> IOS XE 16.x부터 **DHCP Snooping → DAI → IP Source Guard** 3단계 보안 체계가 표준.  
> IOS XE 17.x에서 MACsec으로 L2 트래픽 암호화까지 가능.
