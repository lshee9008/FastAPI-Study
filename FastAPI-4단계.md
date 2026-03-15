# FastAPI-4단계
## 학습할 키워드들
- SQLAlchemy
- ORM(객체관계매핑)
- 데이터모델링
- 비동기DB연결
- AsyncSession
- aiosqlite
- asyncpg
- CRUD
- Alembic(알렘빅)
- DB마이그레이션
- 스키마버전관리
- autogenerate

# SQLAlchemy 기초

- 데이터베이스는 SQL이라는 언어를 쓰지만, 우리는 파이썬으로 코딩 중
- 이때 파이썬 클래스(객체)를 만들면 알아서 SQL 테이블과 매핑(연결)해 주는 기술을 **ORM (Object-Relation Mapping)** 이라고 함
- 그중 파이썬 생태계 1대장 → SQLAlchemy

```python
# models.py
from sqlalchemy import Column, Integer, String
from sqlalchemy.orm imprt declarative_base

# 모든 모델의 뼈대가 될 Base 클래스 생성
Base = declarative_base()

# 파이썬 클래스가 곧 DB의 테이블이 됨
class User(Base):
	__tablename__ = "users" # 실제 DB에 생성될 테이블 이름
	
	id = Column(Integer, primary_key=True, index=True)
	email = Column(String, unique=True, index=True)
	hashed_password = Column(String)
```

- 장점: 복잡한 SQL 쿼리문을 직업 작성할 필요 없이, 파이썬 객체를 다루듯 데이터를 조작할 수 있음

# 2. 비동기 DB 연결 설정

- FastAPI는 요청을 기다리는 동안 다른 일을 처리할 수 있는 ‘비동기(Async)’ 프레임워크이다.
- DB에 데이터를 요청하고 기다리는 시간도 비동기로 처리해야 진정한 성능이 나옴
- (여기서는 테스트가 가장 쉬운 aiosqlite를 예시로 들지만, 실무에서는 asyncpg (PostgreSQL)나 aiomysql(MySQL)을 사용.

```python
# database.py
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker

#비동기 SQLite DB 주소 (실무: "postgresql+asyncpg://user:pass@localhost/dbname")
SQLALCHEMY_DATABASE_URL = "sqlite+aiosqlite:///./test.db"

# 비동기 엔진 및 세션 메이커 생성
engine = create_async_engine(SQLALCHEMY_DATABASE_URL, echo=True)
AsyncSessionLocal = async_sessionmaker(engine, expire_on_commit=False)

# 3단계에서 배운 '의존성 주입(Depends)'을 위한 DB 세션 생성 함수
async def get_db():
	async with AsyncSessionLocal() as session:
		yield session
```

# 3. CRUD 구현 (비동기 방식)

- 3단계에서 배운 Depends를 활용해 DB 세션을 주입받고, C(생성), R(읽기), U(수정), D(삭제) API를 만들어 볼 것
- SQLAlchemy 2.0 비동기 문법을 사용

```python
# main.py
from fastapi import FastAPI, Depends
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from database import get_db, engine
from models import Base, User
from pydantic import BaseModel

app = FastAPI()

# 요청 받을 Pydantic 스키마
class UserCreate(BaseModel):
	email: str
	password: str
	
# 데이터 생성 (Create)
@app.post("/users/")
async def create_user(user: UserCreate, db: AsyncSession = Depends(get_db):
	# 1. ORM 객체 생성
	db_user = User(email=user.email, hashed_password=user.password + "hash")
	# 2. 세션에 추가하고 DB에 커밋(저장)
	db.add(db_user)
	await db.commit()
	await db.refresh(db_user) # 생성된 id값을 가져오기 위해 새로고침
	return db_user
	
# 데이터 읽기 (Read)
@app.get("/users/{user_id}")
async def read_user(user_id: int, db: AsyncSession = Depends(get_db)):
	# 비동기 select 문법 사용
	result = await db.execute(select(User).where(User.id == user_id))
	user = result.scalars().first()
	return user

```

# 4. 데이터베이스 마이그레이션 (Alembic)

- 파이썬 코드(models.py)dptj User 클래스에 nickname이라는 컬럼을 새로 추가했다고 가정
- 파이썬 코드만 바꿨다고 해서 이미 만들어진 DB 테이블이 자동으로 바뀌지는 않음
- 이때 사용하는 것이 Alembic
    - 데이터베이스 스키마(구조)의 버전 관리 시스템(Git 같은 역할) 이라고 생각

### 기본적인 사용 흐름 (터미널 명령어):

1. **alemvic init alembic**: 프로젝트에 alembic 초기 설정 폴더를 만든다.
2. alembic.ini와 [env.py](http://env.py) 파일에서 내 DB 주소와 models.Base를 연결해 줌
3. alembic revision —autogenerate -m “add nickname column”: 파이썬 모델과 현재 DB를 비교해서 변경 사항(마이그레이션 스크립트)을 자동 생성
4. alembic upgrade head: 생성된 스크리브를 실제 DB에 적용
