# ㄴFastAPI-2단계ㄴㄴ
## 학습할 키워드들
- 데이터검증
- Field
- Path
- Query
- 유효성검사
- ResponseModel(응답모델)
- 데이터필터링
- HTTPException
- StatusCode(상태코드)
- 예외처리

# 1. Request Body (요청 본문) 및 Pydantic 모델

- 게시글을 작성하거나 회원가입을 할 때는 URL에 데이터를 담기엔 너무 길고 복잡
- 이때는 데이터를 **Request Body(요청 본문)에 JSON 형태로 담아 보냄**
- FastAPI는 Pydantic이라는 라이브러리를 사용해 이 JSON 데이터를 파이썬 클래스로 깔끔하게 바꿔줌

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

# 1. Pydantic의 BaseModel을 상속받아 데이터 구조를 정의
class Item(BaseModel):
	name: str
	description: str | None = None # | None은 선택 사항(Optional)이라는 뜻
	price: float
	tax: float | None = None

# 2. 만든 모델을 함수의 매개변수 타입으로 지정
@app.get("/items/")
def create_item(item: Item)
	# 이제 item은 단순한 딕셔너리가 아니라, name, price 등의 속성을 가진 파이썬 객체
	total_price = item.price + (item.tax if item.tax else 0)
	return {"itme_name": item.name, "total_price": total_price}
```

- 장점
    - 클라이언트가 price에 솟자 대신 “비싸요”라는 문자열을 보내면
    - FastAPI가 알아서 422 Unprocessable Entity 에러를 뱉어내며 데이터를 팅겨냄
    - 개발자가 일일이 타입 검사 코드를 잘 필요가 없음.

## 2. 데이터 검증 (세부 조건 설정하기)

- 단순히 ‘문자열이다’, ‘숫자다’를 넘어서, “이름은 최소 2글자 이상이어야 해”, “가격은 0보다 커야 해” 같은 디테일한 조건을 걸 수 있음
    - **Field**: Pydantic 모델 내부의 변수를 검증할 때 사용
    - **Query**: 쿼리 매개변수를 검증할 때 사용
    - **Path**: 경로 매개변수를 검증할 때 사용

```python
from fastapi import FastAPI, Path, Query
from pydantic import BaseModel, Field

app = FastAPI()

class User(BaseModel):
	# Field를 사용해 세부 조건 추가 (min_length: 최소 길이, max_length: 최대 길이)
	username: str = Field(..., min_length=3, max_length=50, description="사용자 이름")
	age: int = Field(..., ge=0, le=120, description="나이 (0 이상 120 이하)") # ge: >=, le: <=
	
# Path와 Query를 사용한 검증
@app.get("/users/{user_id}")
def get_user(
	# Path: user_id는 1 이상이어야 함 (ge=1)
	user_id: int = Path(..., ge=1, title="유저 ID")
	# Query: q는 최대 10글자인 선택적 검색어
	q: str | None = Query(Npne, max_length=10)
):
	return {"user_id": user_id, "query": q}
```

# 3. 응답 모델 (Response Model)

- 데이터를 받을 때뿐만 아니라, **클라이언트에게 데이터를 돌려줄 때(응답)** 도 형태를 강제할 수 있음
- 가장 큰 이유는 보안과 필터링
- 데이터베이스에서 회원 정보를 가져왔는데, 실수로 비밀번호까지 클라이언트에게 보내면 큰일 나는 그러한 사례

```python
class UserIn(BaseModel):
	username: str
	password: str
	email: str
	
class UserOut(BaseModel):
	username: str
	email: str
	# password 필드는 제외했음.
	
# response_model에 응답용 스키마(UserOut)를 지정
@app.post("/users/register", response_model=UserOut)
def register_user(user: UserIn):
	# (여기에 DB 저장하는 로직이 있다고 가정)
	# user 객체에는 password가 있지만,
	# FastAPI가 USerOut 모델을 보고 알아서 password를 걸러내고 반환
	return user
```

# 4. 오류 처리 (HTTPException)

- 클라이언트가 잘못된 데이터를 보낸 것은 아니지만, 비즈니스 로직상 문제가 있을 때
- 예: 조회하려는 데이터가 DB에 없을 때, 권한이 없을 때) 상태 코드를 제어하며 에러를 발생시키는 방법

```python
from fastapi import FastAPI, HTTPException

app = FastAPI()

fake_db = {"1": "사과", "2": "바나나"}

@app.get("/items/{item_id}")
def read_item(item_id: str):
	if item_id not in fake_db:
		# HTTPException을 발생시켜 404 Not Found 에러를 클라이언트에게 보냄
		raise HTTPException(status_code=404, detail="해당 아이템을 찾을 수 없습니다.")
		
	return {"item": fake_db[item_id]}
```
