# 08 | 실무에서 꼭 필요한 보안 지식

## SQL Injection
- SQL Injection은 사용자 입력값을 통해 SQL 쿼리를 조작하여 데이터베이스를 공격하는 기법임.
- 공격자가 의도하지 않은 SQL 명령을 실행시켜 데이터를 탈취하거나 삭제, 수정할 수 있음.
- 예를 들어 로그인 폼에서 `admin' OR '1'='1` 같은 입력으로 인증을 우회할 수 있음.
- Prepared Statement를 사용하면 사용자 입력을 쿼리와 분리하여 안전하게 처리할 수 있음.
- ORM(Object-Relational Mapping)을 사용하면 직접 SQL을 작성하지 않아 SQL Injection 위험이 크게 줄어듦.
- 입력값 검증(Validation)과 화이트리스트 방식의 필터링도 중요한 방어책임.
- [SQL 인젝션 모의사이트](https://los.rubiya.kr)

## XSS (Cross Site Scripting)
- XSS는 Cross Site Scripting의 약자인데, CSS는 이미 사용 중이라 XSS로 표기함.
- SQL Injection은 데이터베이스를 공격하는 반면, XSS는 같은 웹 서비스를 사용하는 다른 사용자를 공격 대상으로 함.
- 주로 JavaScript 코드를 삽입하여 사용자의 쿠키, 세션, 개인정보를 탈취하거나 악성 행위를 수행함.
- 사용자 입력값이 프론트엔드 코드에 그대로 반영될 때 발생할 수 있음.

### XSS 종류
- **Stored XSS (저장형)**: 악성 스크립트가 서버 데이터베이스에 저장되어 다른 사용자가 해당 페이지를 볼 때마다 실행됨. 게시판, 댓글 등에서 주로 발생함.
- **Reflected XSS (반사형)**: 악성 스크립트가 URL 파라미터나 폼 입력을 통해 즉시 반사되어 실행됨. 피싱 링크를 통해 주로 유포됨. (링크를 클릭산 사용자만 피해)
- **DOM-based XSS**: 서버를 거치지 않고 클라이언트 측 JavaScript에서 DOM 조작 시 발생함. `innerHTML`, `document.write()` 등 사용 시 주의가 필요함.

### XSS 방어
- HTML entity 인코딩을 통해 특수문자를 변환함 (예: `<` → `&lt;`, `>` → `&gt;`).
- Content Security Policy(CSP)를 설정하여 허용된 출처의 스크립트만 실행되도록 제한함.
- 프레임워크의 자동 이스케이핑 기능을 활용함 (React의 JSX, Vue의 템플릿 등).
- 사용자 입력을 신뢰하지 않고 항상 검증 및 sanitize(정제) 처리를 해야 함.
- [XSS 모의사이트](https://xss-game.appspot.com)

## CSRF (Cross-Site Request Forgery)
- CSRF는 사용자가 의도하지 않은 요청을 공격자가 강제로 실행시키는 공격임.
- 사용자가 로그인된 상태에서 악성 사이트를 방문하면, 사용자의 권한으로 의도치 않은 요청(계좌이체, 정보수정 등)이 실행됨.
- SQL Injection, XSS와 함께 OWASP Top 10에 포함되는 주요 웹 취약점임.

### CSRF 방어
- CSRF 토큰을 사용하여 정상적인 요청인지 검증함. 서버가 발급한 토큰이 포함되지 않은 요청은 거부함.
- SameSite 쿠키 속성을 설정하여 크로스 사이트 요청 시 쿠키 전송을 제한함.
- 중요한 작업에는 재인증(비밀번호 확인 등)을 요구함.
- Referer 헤더 검증을 통해 요청의 출처를 확인함.

## WAF (Web Application Firewall)
- 책에서는 SQL Injection, XSS 등의 공격을 웹 방화벽(WAF)을 통해서 막을 수 있다고 설명함.
- 대표적인 예로는 Cloudflare, AWS WAF 등이 있음.
- 인터넷과 웹 애플리케이션 사이에 배치되어 **역방향 프록시** 방식으로 작동함.
- 클라이언트의 HTTP/HTTPS 요청이 서버에 도달하기 전에 WAF를 먼저 거치게 됨.
- WAF는 애플리케이션 레벨(OSI 모델 7계층)의 공격 패턴을 탐지하고 차단하는 보안 솔루션임.
- 미리 정의된 규칙과 머신러닝을 통해 악성 트래픽을 식별함.
- 일반 방화벽은 네트워크 레벨(3-4계층)에서 IP/포트를 기준으로 차단하지만, WAF는 HTTP 요청 내용을 분석함.
- 하지만 WAF만으로는 완벽한 방어가 불가능하므로 안전한 코딩 습관과 병행해야 함.

### 역방향 프록시
- 일반 연결 : 클라이언트 ➡️ 서버
- 포워드 프록시 : 클라이언트 ➡️ 프록시 서버 ➡️ 서버 / 클라이언트 측에서 설정 및 관리(회사 프록시, VPN)
- 역방향 프록시 : 클라이언트 ➡️ 프록시 서버 ➡️ 서버 / 서버 측에서 설정 및 관리(Cloudflare, Nginx)

## 참고자료
- <누구나 쉽게 따라 하며 배우는 웹 해킹 첫걸음> 권현준 저
- [CLOUDFLARE, WAF란?](https://www.cloudflare.com/ko-kr/learning/ddos/glossary/web-application-firewall-waf/)