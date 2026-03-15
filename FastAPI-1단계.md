# FastAPI-1단계
## 학습할 키워드들

- TypeHint
- 라우팅
- HTTP메서드
- GET, POST, PUT, DELETE
- Path_Parameter(경로매개변수), Query_Parameter(쿼리매개변수

# 1. 환경 세팅 : FastAPU 및 Uvicorn 설치

```bash
pip install fastapi uvicorn
```

- FastAPI는 프레임워크 자체이고, 이를 실행해 줄 ‘웹 서버(ASGI 서버)’가 따로 필요
- 그 역할을 하는 것이 Uvicorn

# 2. 첫 API 만들기

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
def read_root():
	return {"message": "Hello, FastAPI World!"}
```

- 서버 실행하기
    
    ```bash
    uvicorn main:app --reload
    ```
    
    - **main**: 파일명(main.py)
    - **app**: 코드 안에서 만든 FastAPI 인스턴스 이름
    - **—reload**: 코드를 수정하고 저장하면 서버가 자동으로 재시작되는 아주 편리한 옵션(개발 중에만 사용)
    
    ---
    
    - [**http://localhost:8000**](http://localhost:8000) 접속

# 3. 라우팅 기초 (HTTP 메서드 이해)

- 인터넷에서 데이터를 주고 받을 때는 목적에 맞는 행동(메서드)을 정해줌, 대표적인 4가지 구현
    
    ```python
    @app.get("/items")
    def get_items():
    	return {"message": "데이터를 읽어옴 (Read)"}
    	
    @app.post("/items")
    def create_item():
    	return {"message": "새로운 데이터를 생성 (Create)"}
    	
    @app.put("/items")
    def update_item():
    	return {"message": "기존 데이터를 수정 (Update)"}
    	
    @app.delete("/items")
    def delete_item():
    	return {"message": "데이터를 삭제 (Delete)"}
    ```
    
    - **@app.get(…)** 과 같은 부분을 데코레이터라고 부름
    - 어떤 URL 경로로, 어떤 메서드가 들어왔을 때 아래 함수를 실행할지(라우팅) 결정

# 4. 매개변수 (Path vs Query Parameters)

- 클라이언트가 서버로 데이터를 넘겨줄 때 URL을 활용하는 두 가지 방법

### A. 경로 매개변수 (Path Parameters)

- URL 경로 자체에 값을 끼워 넣는 방식
- 주로 특정 자원(데이터) 하나를 정확히 지칭할 때 사용

```python
# /users/123 또는 /users/456 처럼 접속
@app.get("/users/{user_id}")
def get_user(user_id: int):
	# 타입 힌트로 int로 주었기 때문에, /users/abc 로 접속하면 자동으로 에러를 내뿜어 줌
	return {"user_id": user_id, "message": "특정 유저의 정보입니다."}
```

### B. 쿼리 매개변수 (Query Parameters_

- URL 끝에 ?를 붙이고 키=값 형태로 전달하는 방식
- 여러 개일 경우 &로 연결
- 주로 데이터베이스에서 검색, 필터링, 페이징을 할 때 사용

```python
# /search?keyword=python&limit=10 처럼 접속
@app.get("/search")
def search_items(keyword: str, limit: int = 5):
	# limit: int = 5 는 기본값을 5로 설정한다는 뜻 (limit를 안 적으면 5로 작동)
	return {"검색어": keyword, "가져올 갯수": limit}
```
