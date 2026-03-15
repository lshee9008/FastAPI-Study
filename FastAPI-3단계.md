# FastAPI-3단계
## 학습할 키워드들
- APIRouter
- 라우터분리
- 모듈화
- 엔드포인트관리
- 의존성주입
- DependencyInjection(DI)
- Depends
- 공통로직처리
- 미들웨어(Middleware)
- 요청가로채기
- 헤더추가
- CORS
- CORSModdleware
- 교차출처리소스공유

# 1. 라우터 분리(APIRouter): 파일 쪼개기

- 유저 관리 API, 게시글 관리 API, 결제 API .. 기능이 많아질수록 [main.py](http://main.py)가 수천 줄로 길어짐
- 이를 방지하기 위해 APIRouter를 사용하여 도메인(주제)별로 파일을 예쁘게 나눔

### A. 라우터 파일 작성 (예: routers/users.py)

```python
# routers/users.py
from fastapi import APIRouter

#APIRouter 인스턴스 생성 (prefix를 지정하면 URL 앞에 공통으로 붙음)
router = APIRouter(prefix="/users", tags=["Users"])

@router.get("/")
def read_users():
	return [{"username": "Alice"}, {"username": "Bob"}]

@router.get("/{user_id}")
def read_user(user_id: int):
	return {"user_id": user_id, "username": "Alice"}
```

### B. 메인 파일에 연결하기(main.py)

```python
# main.py
from fastapi import FastAPI
from routers import users # 만든 라우터를 불러옴

app = FastAPI()

# app에 라우터를 찰싹 붙여줌
app.include_router(users.router)

@app.get("/")
def root():
	return {"message": "메인 서버가 실행 중입니다."}
```

- **결과**: /users/, /users/1 과 같은 URL이 정상 작동하며, Swagger UI(/docs)에서도 Users 태그로 깔끔하게 그룹화되어 보임

# 2. 의존성 주입 (Dependency Injection): 중복 코드 제거

- FastAPI의 필살기!!!
- 여러 API에서 공통으로 필요한 작업(예: 데이터베이스 연결, 로그인 토큰 검사, 공통 쿼리 파라미터 확인 등)을 함수로 만들어두고, 필요한 곳에 ‘주입’해 줌

```python
from fastapi import FastAPI, Depends, HTTPException

app = FastAPI()

# 1. 공통 로직을 처리할 '의존성 함수'를 만듬
def verify_token(token: str):
	if token != "supersecrettoken":
		raise HTTPException(status_code=400, detail="유효하지 않은 토근입니다.")
	return {"user": "admin"} # 토큰이 맞으면 유저 정보를 반환

# 2. API 엔드포인트에서 Depends()를 사용해 주입 받는다.
@app.get("/secure-data")
def get_secure_data(user_info: dict = Depends(verify_token)):
	# verify_token 함수가 먼저 실행되고, 통과하면 그 결과값이 user_info에 들어옴
	return {"message": "보안 데이터 접근 성공!", "user": user_info}
```

- 장점
    - 코드를 복사해서 붙여넣을 필요가 없음
    - 의존성 함수 하나만 수정하면 이를 주입받은 모든 API에 일괄 적용

# 3. 미들웨어 및 CORS: 전역 처리와 통행증

- API가 하나하나가 아니라, 서버에 들어오고 나가는 모든 요청과 응답을 가로채서 처리해야 할 때 사용

### A. 미들웨어 (Middleware)

- 모든 요청에 대해 실행 시간(Log)을 측정하거나, 특정 헤더를 추가하는 등의 문지기 역할을 함

```python
from fastapi import FastAPI, Request
import time

app = FastAPI()

# 모든 요청을 가로채는 미들웨어
@app.middleware("http")
async def add_process_time_header(request: Request, call_next):
	start_time = time.time()
	
	# call_next(request)가 실제 API 엔드포인트로 요청을 넘겨주는 역할
	response = await call_next(request)
	
	process_time = time.time() - start_time
	
	#응답 헤더에 걸린 시간을 추가해서 보냄
	response.headers["x-Process-Time"] = str(process_time_
	return response
```

### CORS (Cross-Origin Resource Sharing) 설정

- 프론트엔드(예: localhost:3000의 React)와 백엔드(localhost:8000의 FastAPI)의 주소(포트)가 다르면,
- 브라우저는 보안상 API 요청을 차단
- 이를 허용해 주는 통행증 발급 설정

```python
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware

app = FastAPI()

# 허용할 프론트엔드 도메인 목록
origins = [
	"http://localhost:3000",
	"https://my-frontend.com"
]

# CORSMiddleware를 앱에 추가
app.add_middleware(
	CORSMiddleware,
	allow_origins=origins, # 허용할 출처
	allow_credentials=True, # 쿠키 등 인증 정보 허용 여부
	allow_methods=["*"], # 허용할 HTTP 메서드 (GET, POST 등 전체 허용)
	allow_headers=["*"], # 허용할 HTTP 헤더 전체 허용
)
```
