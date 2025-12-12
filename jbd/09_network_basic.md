# 10 | 모르면 답답해지는 네트워크 기초

## 1. OSI 7계층과 TCP/IP 4계층

네트워크 통신이 어떻게 이루어지는지 이해하려면 계층 구조를 알아야 함.

### OSI 7계층

**OSI(Open Systems Interconnection) 모델**은 네트워크 통신을 7개 계층으로 나눈 표준 모델임.

| 계층 | 이름 | 역할 | 프로토콜/예시 |
|------|------|------|--------------|
| 7 | 응용(Application) | 사용자와 직접 상호작용 | HTTP, HTTPS, FTP, SMTP |
| 6 | 표현(Presentation) | 데이터 형식 변환, 암호화 | SSL/TLS, JPEG, MPEG |
| 5 | 세션(Session) | 연결 관리 및 유지 | NetBIOS, SSH |
| 4 | 전송(Transport) | 데이터 전송 방식 결정 | TCP, UDP |
| 3 | 네트워크(Network) | 경로 설정 및 주소 지정 | IP, ICMP, Router |
| 2 | 데이터 링크(Data Link) | 물리적 연결 관리 | Ethernet, MAC, Switch |
| 1 | 물리(Physical) | 실제 데이터 전송 | 케이블, 허브 |

### TCP/IP 4계층

OSI 7계층을 단순화한 버전임.

| 계층 | 이름 | OSI 대응 | 역할 |
|------|------|---------|------|
| 4 | 응용(Application) | 5-7계층 | 애플리케이션 간 통신 |
| 3 | 전송(Transport) | 4계층 | 프로세스 간 통신 |
| 2 | 인터넷(Internet) | 3계층 | 호스트 간 통신 |
| 1 | 네트워크 인터페이스(Network Interface) | 1-2계층 | 물리적 네트워크 통신 |

> 계층 구조 사용 이유: 각 계층이 독립적으로 동작하기 때문에 특정 계층의 문제가 발생해도 다른 계층에 영향을 주지 않음. 예를 들어, HTTP를 HTTPS로 바꿔도 TCP/IP 계층은 변경할 필요가 없음.

## 2. HTTP와 HTTPS

### HTTP(HyperText Transfer Protocol)

**HTTP**는 웹에서 데이터를 주고받기 위한 프로토콜임.

**특징**
- **무상태성(Stateless)**: 각 요청이 독립적이며 이전 요청의 정보를 유지하지 않음
- **요청-응답 구조**: 클라이언트가 요청하면 서버가 응답함
- **텍스트 기반**: 사람이 읽을 수 있는 형태로 데이터를 전송함

**HTTP 메서드**
- **GET**: 리소스 조회
- **POST**: 리소스 생성
- **PUT**: 리소스 전체 수정
- **PATCH**: 리소스 일부 수정
- **DELETE**: 리소스 삭제

**HTTP 상태 코드**:
- **2xx (성공)**: 200 OK, 201 Created, 204 No Content
- **3xx (리다이렉션)**: 301 Moved Permanently, 302 Found, 304 Not Modified
- **4xx (클라이언트 오류)**: 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found
- **5xx (서버 오류)**: 500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable

### HTTPS(HTTP Secure)

**HTTPS**는 HTTP에 보안 계층을 추가한 프로토콜임. SSL/TLS를 사용해 데이터를 암호화함.

**HTTPS의 필요성**
- HTTP는 데이터를 평문으로 전송하기 때문에 중간에 누군가 패킷을 가로채면 내용을 볼 수 있음
- 로그인 정보, 개인정보, 결제 정보 등이 노출될 위험이 있음

**HTTPS의 장점**
- **기밀성**: 데이터를 암호화하여 중간에 가로채도 읽을 수 없음
- **무결성**: 데이터가 중간에 변조되지 않았음을 보장함
- **인증**: 서버가 신뢰할 수 있는 서버임을 증명함
- **SEO 이점**: 검색 엔진이 HTTPS 사이트를 더 높게 평가함

**HTTPS의 단점**
- HTTP보다 약간 느림 (암호화/복호화 과정)
- SSL/TLS 인증서 비용 발생 (Let's Encrypt로 무료 발급 가능)

> 대부분의 웹사이트가 HTTPS를 사용함. Chrome 등의 브라우저는 HTTP 사이트를 "안전하지 않음"으로 표시함.

## 3. IP 주소와 포트

### IP 주소(IP Address)

**IP 주소**는 네트워크에 연결된 각 장치를 식별하는 고유한 주소임.

**IPv4**:
- 32비트 주소 체계 (예: 192.168.0.1)
- 약 43억 개의 주소 제공
- 현재 주소가 거의 고갈됨

**IPv6**:
- 128비트 주소 체계 (예: 2001:0db8:85a3:0000:0000:8a2e:0370:7334)
- 약 340간(3.4×10^38)개의 주소 제공

**공인 IP vs 사설 IP**
- **공인 IP**: 인터넷에서 유일한 주소 (전 세계에서 하나)
- **사설 IP**: 로컬 네트워크 내에서만 사용하는 주소
  - 10.0.0.0 ~ 10.255.255.255
  - 172.16.0.0 ~ 172.31.255.255
  - 192.168.0.0 ~ 192.168.255.255

### 포트(Port)

**포트**는 하나의 컴퓨터에서 여러 서비스를 구분하기 위한 번호임.

**포트 번호 범위**
- **0-1023**: 잘 알려진 포트 (Well-Known Ports)
- **1024-49151**: 등록된 포트 (Registered Ports)
- **49152-65535**: 동적 포트 (Dynamic Ports)

**주요 포트 번호**
- **20/21**: FTP (파일 전송)
- **22**: SSH (보안 원격 접속)
- **25**: SMTP (이메일 발송)
- **53**: DNS (도메인 조회)
- **80**: HTTP (웹 서버)
- **443**: HTTPS (보안 웹 서버)
- **3306**: MySQL
- **5432**: PostgreSQL
- **6379**: Redis
- **27017**: MongoDB
- **8080**: 대체 HTTP (개발 서버)

## 4. DNS(Domain Name System)

### DNS란?

**DNS**는 도메인 이름을 IP 주소로 변환해주는 시스템임.

사람은 "www.example.com" 같은 도메인 이름을 기억하기 쉽지만, 컴퓨터는 IP 주소로만 통신할 수 있음. DNS는 이를 중간에서 변환해주는 역할을 함.

### DNS 동작 과정

1. 사용자가 브라우저에 "www.example.com" 입력
2. 브라우저는 로컬 DNS 캐시 확인
3. 캐시에 없으면 DNS 서버에 질의
4. DNS 서버가 IP 주소 반환 (예: 93.184.216.34)
5. 브라우저가 해당 IP 주소로 접속

### DNS 레코드 타입

- **A 레코드**: 도메인 → IPv4 주소
- **AAAA 레코드**: 도메인 → IPv6 주소
- **CNAME 레코드**: 도메인 → 다른 도메인 (별칭)
- **MX 레코드**: 메일 서버 지정
- **TXT 레코드**: 텍스트 정보 (주로 인증용)
- **NS 레코드**: 네임 서버 지정

### DNS의 중요성

**장점**
- 사람이 기억하기 쉬운 도메인 사용 가능
- IP 주소가 바뀌어도 도메인은 그대로 유지 가능
- 여러 서버로 트래픽 분산 가능

**단점**
- DNS 조회 시간이 추가됨 (보통 20-120ms)
- DNS 서버 장애 시 서비스 접속 불가
- DNS 캐시로 인해 IP 변경 시 즉시 반영 안 됨

**DNS 캐싱**
- 브라우저, 운영체제, 라우터, ISP 등 여러 단계에서 캐싱됨
- TTL(Time To Live) 설정으로 캐시 유효 시간 조절
- 성능 향상과 DNS 서버 부하 감소

> DNS 조회는 빠른 응답이 중요하고 작은 데이터를 주고받기 때문에 UDP(53번 포트)를 사용함. 단, 응답 크기가 512바이트를 초과하면 TCP를 사용함.

## 참고 자료

1. [MDN Web Docs, "HTTP 개요", 2024](https://developer.mozilla.org/ko/docs/Web/HTTP/Overview)
2. [Cloudflare, "What is a Load Balancer?", 2024](https://www.cloudflare.com/learning/performance/what-is-load-balancing/)
3. [AWS, "OSI 모델이란?", 2024](https://aws.amazon.com/ko/what-is/osi-model/)