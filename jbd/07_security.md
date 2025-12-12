# 08 | 실무에서 꼭 필요한 보안 지식

## 1. 인증과 인가

웹 애플리케이션 보안의 가장 기본이 되는 두 가지 개념을 이해해야 함.

- **인증(Authentication)**: "당신이 누구인지" 확인하는 과정 (로그인)
- **인가(Authorization)**: "무엇을 할 수 있는지" 결정하는 과정 (권한 확인)

### JWT 기반 인증

```javascript
const jwt = require('jsonwebtoken');

// 토큰 생성
function generateToken(user) {
    return jwt.sign(
        { id: user.id, email: user.email },
        process.env.JWT_SECRET,
        { expiresIn: '1h' }
    );
}

// 토큰 검증
function verifyToken(req, res, next) {
    const token = req.headers['authorization']?.split(' ')[1];
    
    try {
        const decoded = jwt.verify(token, process.env.JWT_SECRET);
        req.user = decoded;
        next();
    } catch (error) {
        return res.status(403).json({ error: 'Invalid token' });
    }
}
```

**세션 vs JWT**: 세션은 서버가 상태를 저장하고 관리하기 쉽지만 확장성이 낮음. JWT는 상태를 저장하지 않아 확장성이 좋지만 토큰을 즉시 무효화할 수 없음.

## 2. SQL 인젝션

악의적인 SQL 문을 주입하여 DB를 조작하는 공격으로, 가장 흔하고 위험한 보안 취약점 중 하나임.

### 취약한 코드 vs 안전한 코드

```python
# 취약한 코드
query = f"SELECT * FROM users WHERE username = '{username}'"

# 안전한 코드
query = "SELECT * FROM users WHERE username = ?"
db.execute(query, (username,))
```

**방어 방법**
- Prepared Statement 사용 (파라미터 바인딩)
- 입력 값 검증 및 이스케이프 처리
- DB 사용자 권한 최소화

## 3. XSS(Cross-Site Scripting)

웹 페이지에 악성 스크립트를 삽입하는 공격으로, 사용자의 쿠키를 탈취하거나 세션을 하이재킹할 수 있음.

```javascript
// 출력 시 이스케이프 처리
function escapeHtml(text) {
    const map = {
        '&': '&amp;',
        '<': '&lt;',
        '>': '&gt;',
        '"': '&quot;',
        "'": '&#039;'
    };
    return text.replace(/[&<>"']/g, m => map[m]);
}

// 사용자 입력을 그대로 출력하지 않기
document.getElementById('output').textContent = userInput; // 안전
// document.getElementById('output').innerHTML = userInput; // 위험!
```

## 4. CSRF(Cross-Site Request Forgery)

사용자가 자신의 의지와 무관하게 공격자가 의도한 행위를 하도록 만드는 공격.

**방어 방법**
- CSRF 토큰 사용
- SameSite 쿠키 속성 설정
- Referer 검증

## 5. 비밀번호 보안

```python
import bcrypt

# 비밀번호 해싱
def hash_password(password):
    salt = bcrypt.gensalt()
    return bcrypt.hashpw(password.encode('utf-8'), salt)

# 비밀번호 검증
def verify_password(password, hashed):
    return bcrypt.checkpw(password.encode('utf-8'), hashed)
```

**문제사항**
- 평문 저장
- MD5, SHA-1 같은 빠른 해시 사용
- Salt 없이 해싱

## 6. 보안 헤더

```javascript
app.use((req, res, next) => {
    res.setHeader('X-XSS-Protection', '1; mode=block');
    res.setHeader('X-Frame-Options', 'DENY');
    res.setHeader('X-Content-Type-Options', 'nosniff');
    res.setHeader('Strict-Transport-Security', 'max-age=31536000');
    next();
});
```

주요 헤더들은 XSS, 클릭재킹, MIME 스니핑 등의 공격을 방어함.

## 7. API 보안

### Rate Limiting

```javascript
const rateLimit = require('express-rate-limit');

const apiLimiter = rateLimit({
    windowMs: 15 * 60 * 1000, // 15분
    max: 100, // 최대 요청 수
    message: 'Too many requests'
});

app.use('/api/', apiLimiter);
```

**추가 보안 사항**
- API 키를 환경 변수로 관리
- HTTPS 필수 사용
- 적절한 CORS 설정

## 8. 환경 변수와 설정 관리

민감한 정보는 절대 코드에 하드코딩하지 않음.

```javascript
// 나쁜 예
const apiKey = "sk_live_abcd1234";

// 좋은 예
const apiKey = process.env.API_KEY;
```

**.env 파일 사용**
- 개발 환경에서는 .env 파일 사용
- .env는 반드시 .gitignore에 추가
- 프로덕션에서는 환경 변수 직접 설정

> **💡 중요**: 모든 사용자 입력은 신뢰할 수 없다고 가정하고, 항상 검증과 이스케이프 처리를 수행해야 함. 보안은 한 번에 끝나는 것이 아니라 지속적으로 관리해야 하는 프로세스임.