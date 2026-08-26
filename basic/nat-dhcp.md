# NAT & DHCP

---

## NAT (Network Address Translation)

사설 IP 주소를 공인 IP 주소로 변환하여 인터넷 통신을 가능하게 하는 기능

### 사설 IP 주소 범위

| 클래스 | 사설 IP 범위 |
|--------|-------------|
| A | 10.0.0.0 ~ 10.255.255.255 |
| B | 172.16.0.0 ~ 172.31.255.255 |
| C | 192.168.0.0 ~ 192.168.255.255 |

### NAT 종류

| 종류 | 설명 |
|------|------|
| **Static NAT** | 사설 IP ↔ 공인 IP 1:1 고정 매핑 (서버 공개에 사용) |
| **Dynamic NAT** | 여러 사설 IP → 공인 IP Pool에서 동적 할당 |
| **PAT (Overload)** | 여러 사설 IP → 1개 공인 IP (Port번호로 구분) — 가장 일반적 |

---

## Static NAT

사설 IP 1개와 공인 IP 1개를 1:1로 영구 매핑 → 외부에서 내부 서버에 직접 접근 가능

```
! 사설 IP 192.168.10.1 ↔ 공인 IP 121.160.23.2 고정 매핑
ip nat inside source static 192.168.10.1 121.160.23.2
!
interface fastethernet 0/1
 ip nat inside       ! 내부(사설) 네트워크 방향 인터페이스
!
interface serial 1/0
 ip nat outside      ! 외부(공인) 네트워크 방향 인터페이스
```

### Static NAT 확인

```
GIT# show ip nat translation
Pro  Inside global     Inside local      Outside local    Outside global
---  121.160.23.2      192.168.10.1      ---              ---
icmp 121.160.23.2:5   192.168.10.1:5   211.241.228.4:5  211.241.228.4:5
```

### Static NAT Debug 확인

```
GIT# debug ip nat
! 내부 → 외부 (NAT 변환)
NAT*: s=192.168.10.1->121.160.23.2, d=211.241.228.4

! 외부 → 내부 (역변환)
NAT*: s=211.241.228.4, d=121.160.23.2->192.168.10.1
```

---

## Dynamic NAT (PAT / Overload)

여러 사설 IP가 1개 공인 IP의 포트 번호로 구분되어 동시 통신 — 가장 널리 사용

```
! 1단계: ACL로 변환 대상 사설 네트워크 정의
access-list 1 permit 192.168.10.0 0.0.0.255

! 2단계: NAT Pool 정의 (공인 IP 범위)
ip nat pool S_NET 121.160.23.2 121.160.23.2 netmask 255.255.255.0

! 3단계: ACL + Pool 연결 (overload = PAT, 포트로 구분)
ip nat inside source list 1 pool S_NET overload

! 4단계: 인터페이스 방향 지정
interface fastethernet 0/1
 ip nat inside
interface serial 1/0
 ip nat outside
```

### PAT 확인

```
GIT# show ip nat translation
Pro  Inside global        Inside local       Outside local      Outside global
icmp 121.160.23.2:11     192.168.10.1:11   211.241.228.4:11   211.241.228.4:11
icmp 121.160.23.2:12     192.168.10.2:12   211.241.228.4:12   211.241.228.4:12
icmp 121.160.23.2:13     192.168.10.3:13   211.241.228.4:13   211.241.228.4:13
```

> PAT: 동일한 공인 IP에 Port 번호만 다르게 여러 사설 주소가 매핑됨

### NAT 항목 수동 삭제

```
GIT# clear ip nat translation *      ! 모든 동적 NAT 항목 삭제
GIT(config)# no ip nat inside source static 192.168.10.1 121.160.23.2  ! Static NAT 삭제
```

### 정보 확인

```
R# show ip nat translation     ! 현재 NAT 테이블 (변환 항목 목록)
R# show ip nat statistics      ! NAT 변환 횟수, 히트/미스 통계
R# debug ip nat                ! 실시간 NAT 변환 패킷 확인
```

---

## DHCP (Dynamic Host Configuration Protocol)

PC나 단말기에게 IP 주소, Subnet Mask, Gateway, DNS 등을 자동으로 할당하는 프로토콜

- UDP **67** (Server), UDP **68** (Client)
- 4단계 메시지: **Discover → Offer → Request → ACK**

### DHCP 4단계 동작

| 단계 | 방향 | 설명 |
|------|------|------|
| **Discover** | Client → Broadcast | DHCP 서버 탐색 |
| **Offer** | Server → Client | IP 주소 제안 (ARP로 중복 확인 후) |
| **Request** | Client → Broadcast | 제안된 주소 사용 요청 |
| **ACK** | Server → Client | 최종 승인 및 정보 전달 |

### DHCP 서버 설정

```
! 특정 주소 DHCP 할당 제외 (서버/게이트웨이 IP는 제외 필수)
ip dhcp excluded-address 192.168.10.250 192.168.10.254

! DHCP Pool 생성
ip dhcp pool CCNA
 network 192.168.10.0 255.255.255.0   ! 할당할 네트워크 대역
 default-router 192.168.10.254        ! 게이트웨이 주소
 dns-server 192.168.10.252 192.168.10.253  ! DNS (Primary, Secondary)
 lease 10                             ! 임대 기간 (일 단위, infinite = 무제한)
```

### DHCP 클라이언트 설정

```
interface fastethernet 0/1
 no shutdown
 ip address dhcp    ! DHCP 클라이언트로 동작 — 자동 IP 수신
```

### 갱신 동작

| 시점 | 동작 |
|------|------|
| 임대 기간 50% 경과 | 동일 서버에 갱신 요청 |
| 임대 기간 87.5% 경과 | 브로드캐스트로 갱신 재시도 |
| 임대 기간 100% 만료 | IP 반납, 재시작 |

### 확인

```
! DHCP 할당된 IP 및 임대 만료 시간 확인
PC# show dhcp lease

! 인터페이스 IP 할당 여부 확인
PC# show ip interface brief

! 서버측에서 할당 현황 확인
Server# show ip dhcp binding

! DHCP 제외 주소 목록 확인
Server# show ip dhcp pool
```

---

## IOS XE 17.x 최신 트렌드

> 출처: [IOS XE 17.x NAT Configuration Guide](https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-addressing/b-ip-addressing/m_iadnat-addr-consv-xe.html)

### NAT — 인터페이스 직접 Overload (`IOS XE 16.x+`)

Pool을 별도로 만들지 않고 외부 인터페이스 IP를 직접 사용하는 방식 — 공인 IP 1개 환경에서 간결

```
! IOS 15.x 기존 방식 — Pool + ACL 2단계
access-list 1 permit 192.168.10.0 0.0.0.255
ip nat pool S_NET 121.160.23.2 121.160.23.2 netmask 255.255.255.0
ip nat inside source list 1 pool S_NET overload

! IOS XE 17.x 간결한 방식 — Pool 없이 인터페이스 직접 지정
access-list 1 permit 192.168.10.0 0.0.0.255
ip nat inside source list 1 interface GigabitEthernet0/1 overload
!                                    ↑ 외부 인터페이스 IP를 공인 IP로 자동 사용
```

### DHCP — NTP 동기화 연동 (`IOS XE 17.x`)

```
! NTP 동기화 이후 DHCP 바인딩 파일에 정확한 임대 시간 기록
! → NTP 설정이 선행되어야 DHCP 임대 정보가 정확히 유지됨
ntp server 1.1.1.1

! DHCP 임대 데이터베이스 파일 저장 (재부팅 후에도 유지)
ip dhcp database flash:/dhcp-bindings.db
```

| 버전 | 비교 |
|------|------|
| **IOS 15.x (Phase 2 자료)** | Pool 방식 PAT, Static/Dynamic NAT, 기본 DHCP |
| **IOS XE 17.x (현재 권장)** | 인터페이스 직접 Overload, DHCP-NTP 연동, DB 파일 저장 |
