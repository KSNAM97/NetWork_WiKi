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
ntp server 211.241.228.2 minpoll 4 maxpoll 10   ! poll 간격 2^4~2^10초
ntp server 211.241.228.2 prefer                  ! 우선 서버 지정

! 큰 시간 변동 방지 (클럭 급변 차단)
ntp panic-threshold apply
```

| 버전 | 비교 |
|------|------|
| **IOS 15.x (Phase 2 자료)** | RSA 1024bit, 기본 MD5, NTPv3 |
| **IOS XE 17.x (현재 권장)** | RSA 2048/4096bit, ECDSA/Ed25519, NTPv4, 알고리즘 명시 설정 |
