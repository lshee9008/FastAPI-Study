# FastAPI-5단계
## 학습할 키워드들
- Passlib
- 비밀번호해싱
- 단방향암호화
- bcrypt
- JWT(JSONWebToken)
- 토큰발급(encode)
- 토큰검증(decode)
- 만료시간(exp)
- OAuth2
- OAuth2PasswordBearer
- 권한 부여(Authorization)
- 의존성주입(Depends)

# 1. 비밀번호 해싱 (Passlib)

- 사용자의 비밀번호를 데이터베이스에 ‘1234’처럼 그대로 저장하는 것은 보안상 절대 금기
- 비밀번호를 암호화하여 저장
- 이때 원래 비밀번호로 되돌릴 수 없는 ‘단방향 암호화(해싱)’방식을 사용
- 파이썬에서는 Passlib (보통 bcrypt 알고리즘과 함께) 라이브러리를 주로 사용

```python
# pip install "passlib[pcrypt]"
from passlib.context import CryptContext

# bcrypt 알고리즘을 사용하는 해싱 컨텍스트 생성
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

# 1. 회원가입 시: 평문 비밀번호를 해시로 변환하여 DB에 저장
def get_password_hash(password: str):
    return pwd_context.hash(password)

# 2. 로그인 시: 입력받은 비밀번호와 DB의 해시가 일치하는지 검증
def verify_password(plain_password: str, hashed_password: str):
    return pwd_context.verify(plain_password, hashed_password)
```

# 2. oAuth2 및 JWT (JSON Web Token) 발급

- 사용자가 아이디와 비밀번호를 맞게 입력했다면, 서버는 “너 인증됐어!”라는 의미로 출입증을 발급
- 최근 백엔드 표준으로 가장 많이 쓰이는 출입증이 바로 **JWT(JSON Web Token)**
- JWT는 클라이언트가 보관하다가 API를 요청할 떄마다 헤더(Header)에 담아서 보냄
- 서버는 이 토큰이 유효한지(위조되지 않았는지, 만료되지 않았는지) 검사만 하면 되므로, 서버에 부하가 적음

```python
# pip install PyJWT
import jwt
from datetime import datetime, timedelta

SECRET_KEY = "super_secret_key_please_change_this" # 절대 외부에 노출되면 안 되는 키!
ALGORITHM = "HS256"

# 로그인 성공 시 클라이언트에게 발급할 토큰 생성 함수
def create_access_token(data: dict, expires_delta: timedelta):
    to_encode = data.copy()
    
    # 만료 시간(exp) 설정
    expire = datetime.utcnow() + expires_delta
    to_encode.update({"exp": expire})
    
    # SECRET_KEY를 사용해 서명(Sign)하여 토큰 발행
    encoded_jwt = jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
    return encoded_jwt
```

# 3. 권한 부여: 의존성 주입(Depends) 활용

- 3단계 에서 배운 **의존성 주입(Depends)** 를 이용
- FastAPI에서 기본 제공하는 OAuth2PasswordBearer를 사용하면, 요청 헤더에서 JWT를 알아서 뽑아오고 검증하는 로직을 아주 우아하게 만들 수 있음

```python
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer

app = FastAPI()

# 클라이언트가 토큰을 받기 위해 요청해야 하는 URL(엔드포인트) 지정
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="login")

# 🔒 모든 보안 API 앞에 서 있을 '문지기' 함수 (의존성)
def get_current_user(token: str = Depends(oauth2_scheme)):
    try:
        # 1. 토큰 해독 시도
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username: str = payload.get("sub") # 보통 sub(subject)에 유저 식별자를 넣습니다.
        if username is None:
            raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="유효하지 않은 토큰")
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="토큰이 만료되었습니다.")
    except jwt.PyJWTError:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED, detail="토큰 검증 실패")
    
    # 2. (실제로는 여기서 DB를 조회해서 유저 객체를 가져옵니다)
    return {"username": username}

# 🛡️ 로그인한 사용자만 접근할 수 있는 보호된 API
@app.get("/users/me")
def read_users_me(current_user: dict = Depends(get_current_user)):
    # get_current_user에서 에러가 나면 여기까지 오지도 못하고 401 에러를 뱉습니다.
    # 여기까지 왔다는 것은 인증된 사용자라는 뜻!
    return {"message": "당신의 개인 정보입니다.", "user": current_user}
```
