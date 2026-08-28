# NTP & SSH

---

## NTP (Network Time Protocol)

네트워크 장비 간 시간을 동기화하는 프로토콜

- UDP **123** 포트
- 초기 동기화 후 3회는 **45분** 간격, 이후 **8시간** 간격으로 동기화
- 인증 기능 지원 (MD5)

### NTP Stratum (계층 구조)

| Stratum | 설명 |
|---------|------|
| 0 | 세슘 원자시계, GPS, 라디오 시계 |
| 1 | Stratum 0에 직접 연결된 NTP 서버 |
| 2 | Stratum 1로부터 동기화된 서버 |
| 3+ | 단계가 내려갈수록 +1 증가 |
| **16** | 동기화되지 않은 상태 |

---

## NTP 설정

### 시간 수동 설정

```
! 현재 시간 확인
R1# show clock
*00:03:54.051 UTC Fri Mar 1 2002    ← * = 동기화되지 않은 상태

! 시간 직접 설정
R1# clock set 13:00:00 5 jun 2026

R1# show clock
13:00:12.199 UTC Fri Jun 5 2026
```

### NTP Server 설정 (R1)

```
! NTP 트래픽 송신에 사용할 인터페이스 지정
ntp source loopback 211

! MD5 인증 설정
ntp authenticate                             ! 인증 기능 활성화
ntp authentication-key 20260605 md5 SOL_NTP  ! Key ID + 패스워드
ntp trusted-key 20260605                     ! 신뢰할 Key 지정

! 이 장비를 NTP Master로 선언 (Stratum 2)
ntp master 2
```

### NTP Client 설정 (R2, R3)

```
ntp authenticate
ntp authentication-key 20260605 md5 SOL_NTP  ! 서버와 동일한 Key
ntp trusted-key 20260605
ntp server 211.241.228.2 key 20260605         ! NTP Server IP + Key 지정
```

> Client의 Stratum = Master의 Stratum + 1 → `ntp master 2`이면 Client는 Stratum 3

### 시간대 설정 (한국 표준시)

```
! UTC +9시간 (KST = Korea Standard Time)
R(config)# clock timezone KST +9

R# show clock
22:38:28.727 KST Fri Jun 5 2026
```

### NTP 확인

```
! NTP 동기화 상태 확인 — * 없으면 동기화 완료
R# show clock

! NTP Server/Client 연결 상태 및 Stratum 확인
R# show ntp status

! NTP 이웃 목록 및 동기화 상태 확인
R# show ntp associations
```

---

## Telnet

원격지의 Router, Switch, Server에 접속하여 장비를 관리하는 원격 접속 프로토콜

- TCP **23** 포트
- 모든 데이터(사용자 계정, 패스워드, 명령어, 출력 결과)가 **평문 전송** — 보안 취약
- 운영 환경에서는 SSH 사용 권장

### Privilege Level (권한 레벨)

| Level | 설명 |
|-------|------|
| 0 | 제한된 기본 명령어만 사용 |
| 1 | 일반 User EXEC Mode |
| 2 ~ 14 | 관리자가 지정한 일부 명령어 |
| **15** | 최고 관리자 권한 (모든 기능) |

### Telnet 기본 설정

```
! VTY 라인 패스워드 설정 (단순 패스워드 방식)
R(config)# line vty 0 4
R(config-line)# password ciscovty
R(config-line)# login
!
R(config)# enable secret cisco    ! Privilege Mode 전환 패스워드
```

### Telnet 접속

```
R3# telnet 1.1.1.1
Trying 1.1.1.1 ... Open

User Access Verification
Password:
R1>
R1> enable
Password:
R1#
```

### 사용자 계정별 Privilege Level 설정

```
! 계정 생성 + Privilege Level 부여
R1(config)# username guest privilege 2 password guest1234
R1(config)# username admin privilege 15 password cisco
!
R1(config)# line vty 0 4
R1(config-line)# login local    ! 로컬 username/password로 인증
```

> 기본 상태에서는 Privilege Level 15를 제외한 나머지 Level(2-14)은 실제로 사용할 수 있는 명령어가 거의 동일하거나 제한적이다. `privilege 3`, `privilege 5`처럼 Level만 다르게 계정을 만든다고 자동으로 서로 다른 권한이 부여되는 것은 아니며, 각 Level별로 실행 가능한 명령어를 관리자가 직접 지정해야 한다.

### Level별 명령어 권한 세분화 (`privilege` 명령어)

관리자 계정 또는 Console로 접속하여 특정 Privilege Level에 명령어 실행 권한을 개별적으로 부여할 수 있다.

```
! guest(Level 2) 계정이 Global Mode(configure terminal)로 진입 가능하도록 허용
R1(config)# privilege exec level 2 configure
R1(config)# privilege exec level 2 configure terminal

! Global Mode에서 interface 명령어 사용 허용
R1(config)# privilege configure level 2 interface

! Interface Mode에서 ip address 명령어 사용 허용
R1(config)# privilege interface level 2 ip
R1(config)# privilege interface level 2 ip address

! Global Mode에서 router rip 및 하위 명령어(version, network) 사용 허용
R1(config)# privilege configure level 2 router
R1(config)# privilege router level 2 version
R1(config)# privilege router level 2 network
```

- 권한을 부여하기 전에는 `% Invalid input detected at '^' marker.` 오류로 해당 명령어가 차단된다.
- **상위 Level은 하위 Level의 권한을 그대로 사용할 수 있다** — 예를 들어 Level 3 계정은 Level 2에 부여된 권한(configure, interface, router rip 등)을 모두 사용 가능하다.

### Autocommand (자동 명령 실행)

특정 계정으로 Telnet/SSH 접속 시 사용자가 직접 명령어를 입력하지 않아도 미리 지정한 명령어가 로그인 직후 자동 실행되고, 실행이 끝나면 세션이 자동 종료되는 기능. 특정 사용자에게 전체 CLI 권한을 주지 않고 정해진 명령어 하나만 실행시킬 때 사용한다.

```
! R1
username admin privilege 15 password 0 cisco
username guest privilege 2 password 0 guest1234
username guest2 privilege 3 password 0 guest4321
username guest2 autocommand show ip route
!
line vty 0 4
 login local
```

guest2 계정으로 Telnet 접속하면 로그인 직후 `show ip route` 결과가 출력되고 곧바로 `[Connection to 1.1.1.1 closed by foreign host]`로 세션이 끊어진다.

### Autocommand Menu (메뉴 기반 제한 접속)

일반 CLI 대신 관리자가 미리 구성한 Menu 화면을 보여주고, 사용자는 번호(Key)만 선택해 정해진 명령어만 실행하는 기능. `menu` 명령어로 Menu Text(화면에 표시될 항목)와 Menu Command(선택 시 실행될 실제 명령어)를 구성한 뒤 계정의 autocommand에 연결한다.

```
R1(config)# menu sol text 1 Routing-Table
R1(config)# menu sol command 1 show ip route
R1(config)# menu sol text 2 Routing-table-eigrp
R1(config)# menu sol command 2 show ip route eigrp
R1(config)# menu sol text 3 TCP-Brief
R1(config)# menu sol command 3 show tcp brief
R1(config)# menu sol text 4 Interface-Brief
R1(config)# menu sol command 4 show ip interface brief
R1(config)# menu sol text 5 R2_Loopback_ICMP
R1(config)# menu sol command 5 ping 2.2.2.2
R1(config)# menu sol text 6 EXIT
R1(config)# menu sol command 6 exit
!
R1(config)# username guest3 privilege 4 password guest4444
R1(config)# username guest3 autocommand menu sol
!
R1(config)# line vty 0 4
R1(config-line)# login local
```

### Banner 설정

접속 시 경고/안내 문구(무단 접속 금지, 관리자 연락처 등)를 출력하는 기능. Telnet, SSH, Console, AUX 접속 모두에 적용된다.

```
R1(config)# banner motd *
Enter TEXT message.  End with the character '*'.
##############################################
                WARNING
##############################################
Name : Ryu Chang Wan
Phone : 010-1234-5678
##############################################
*
```

접속 시 `User Access Verification` 프롬프트보다 먼저 배너 문구가 출력된다.

### Telnet Port 변경 (rotary)

VTY Line의 `rotary` 기능을 사용하면 기본 TCP 23번이 아닌 다른 포트로 Telnet 접속을 받도록 변경할 수 있다. `rotary`는 TCP 3000번을 기준으로 동작 — `rotary 40`이면 TCP 3040번 포트가 된다.

```
! TCP 3040번 포트로만 Telnet 접속을 허용
username admin privilege 15 password cisco
!
access-list 101 permit tcp any any eq 3040
!
line vty 0 4
 login local
 rotary 40
 access-class 101 in
```

확인: `R3# telnet 1.1.1.1 3040`처럼 포트 번호를 명시해야 접속되며, 기본 포트(23)로는 연결되지 않는다.

**예시 문제**: R1의 VTY 라인에는 아무런 접속 제한이 없어서 아무 IP에서나 Telnet 접속을 시도할 수 있다. 관리 목적으로 오직 3.3.3.3(관리자 PC/Loopback)에서만 접속을 허용하고 나머지는 모두 거부하려면 어떻게 해야 하는가?

```
R1(config)# access-list 1 permit 3.3.3.3 0.0.0.0   ! 허용할 출발지 IP만 정확히 매칭
!
R1(config)# line vty 0 4
R1(config-line)# access-class 1 in                  ! VTY 인바운드 접속에 ACL 1 적용
```

확인: 허용되지 않은 IP에서 `R3# telnet 1.1.1.1`을 시도하면 `% Connection refused by remote host`로 즉시 거부되지만, `R3# telnet 1.1.1.1 /source-interface loopback 0`처럼 출발지를 3.3.3.3으로 지정하면 정상 접속된다. `ip access-group`(인터페이스용)와 달리 VTY 라인에는 반드시 `access-class`로 적용해야 한다는 점에 주의.

---

## SSH (Secure Shell)

Telnet의 보안 취약점(평문 전송)을 보완한 암호화 원격 접속 프로토콜

- TCP **22** 포트
- 사용자 계정/패스워드/명령어 모두 **암호화** 전송
- RSA 키 기반 암호화 (Hostname + Domain Name 조합으로 키 생성)

### Telnet vs SSH 비교

| 항목 | Telnet | SSH |
|------|--------|-----|
| 보안 | 평문 (취약) | 암호화 |
| 포트 | TCP 23 | TCP 22 |
| 인증 | 패스워드 | 패스워드 / Public Key |
| 권장 | ❌ | ✅ |

---

## SSH 설정

```
! 1단계: Hostname과 Domain Name 설정 (RSA 키 생성에 필요)
R3(config)# hostname R3
R3(config)# ip domain-name soldesk.com

! 2단계: 로컬 계정 생성
R3(config)# username admin privilege 15 password cisco

! 3단계: RSA 키 생성 (키 이름 = Hostname.DomainName)
R3(config)# crypto key generate rsa
! → The name for the keys will be: R3.soldesk.com
! → How many bits in the modulus [512]: 1024   ← 최소 1024bit 권장

! 4단계: VTY 라인에 로컬 인증 적용
R3(config)# line vty 0 4
R3(config-line)# login local   ! 로컬 username/password로 인증
```

### SSH 접속 명령어

```
! 기본 SSH 접속
R1# ssh -l admin 3.3.3.3

! 암호화 알고리즘 지정 접속
R1# ssh -c aes128-cbc -m hmac-sha1-96 -l admin 3.3.3.3

! 옵션 설명
! -c : 암호화 알고리즘 (aes128-cbc, aes256-cbc 등)
! -m : 해시/인증 알고리즘 (hmac-sha1, hmac-sha1-96 등)
! -l : 로그인 계정명
```

### SSH 버전 지정 (IOS-XE 권장)

```
! SSHv2만 허용 (v1은 보안 취약점 있음)
R(config)# ip ssh version 2

! SSH 타임아웃 및 재시도 설정
R(config)# ip ssh time-out 60       ! 연결 대기 시간 (초)
R(config)# ip ssh authentication-retries 3  ! 인증 실패 허용 횟수
```

### 확인

```
! SSH 버전 및 설정 확인
R# show ip ssh

! 현재 활성 SSH 세션 확인
R# show ssh
```

---

## IOS XE 17.x 최신 트렌드

> 출처: [SSH Algorithms for Common Criteria — IOS XE 17.9+](https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9500/software/release/17-9/configuration_guide/sec/b_179_sec_9500_cg/ssh_algorithms_for_common_criteria_certification.html)

### SSH — RSA 키 크기 현대화 (`IOS XE 17.x+`)

```
! IOS 15.x 기존 방식 — 512bit(기본), 1024bit
crypto key generate rsa
How many bits in the modulus [512]: 1024

! IOS XE 17.x 권장 — 2048bit 이상 (SSHv2 필수 요건)
crypto key generate rsa modulus 2048    ! 2048bit 권장 (FIPS 준수)
crypto key generate rsa modulus 4096    ! 고보안 환경

! 기존 키 삭제 후 재생성
crypto key zeroize rsa
crypto key generate rsa modulus 2048
```

### SSH 알고리즘 명시적 설정 (`IOS XE 17.9+`)

IOS XE 17.10 이상에서 `diffie-hellman-group14-sha1`, `hmac-sha1` 등 레거시 알고리즘이 기본 비활성화됨

```
! 키 교환 알고리즘 지정 (강력한 순서로 나열)
ip ssh server algorithm kex ecdh-sha2-nistp521 ecdh-sha2-nistp384 ecdh-sha2-nistp256

! 호스트 키 서명 알고리즘
ip ssh server algorithm hostkey rsa-sha2-512 rsa-sha2-256

! Public Key 알고리즘 (ECDSA, Ed25519 포함)
ip ssh server algorithm publickey ecdsa-sha2-nistp521 ecdsa-sha2-nistp384 rsa-sha2-512

! MAC 알고리즘
ip ssh server algorithm mac hmac-sha2-256 hmac-sha2-512
```

> ⚠️ Ed25519(`ssh-ed25519`)는 FIPS 모드에서 비지원 — FIPS 환경이면 ECDSA/RSA 사용

### NTP — NTPv4 (`IOS XE 17.x+`)

```
! IOS 15.x 기존 방식
ntp server 211.241.228.2

! IOS XE 17.x 추가 옵션 — 폴링 간격 제어
ntp server 211.241.228.2 minpoll 4 maxpoll 10   ! poll 간격 2^4-2^10초
ntp server 211.241.228.2 prefer                  ! 우선 서버 지정

! 큰 시간 변동 방지 (클럭 급변 차단)
ntp panic-threshold apply
```

| 버전 | 비교 |
|------|------|
| **IOS 15.x (Phase 2 자료)** | RSA 1024bit, 기본 MD5, NTPv3 |
| **IOS XE 17.x (현재 권장)** | RSA 2048/4096bit, ECDSA/Ed25519, NTPv4, 알고리즘 명시 설정 |
