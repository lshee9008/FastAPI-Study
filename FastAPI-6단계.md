# FastAPI-6단계
## 학습할 키워드들
- pytest
- TestClient
- 단위테스트
- 테스트자동화
- Docker
- Dockerfile
- 컨테이너환경
- Gunicorn
- UvicornWorker
- 프로덕션환경

# API 테스트 (pytest 및 TestClient)

- 지금까지는 /docs (Swagger UI)에 들어가서 일일이 버튼을 눌러가며 테스트를 했다.
- 하지만 기능이 100개가 넘어가면 사람이 일일이 테스트할 수 없음
- FastAPI는 내장된 TestClient와 파이썬의 pytest를 활용해 이 과정을 자동화할 수 있게 해줌

```python
# test_main.py
from fastapi.testclient import TestClient
from main import app  # 우리가 만든 FastAPI 앱을 불러옵니다.

client = TestClient(app)

def test_read_root():
    # 클라이언트가 GET "/" 요청을 보낸 것처럼 흉내 냅니다.
    response = client.get("/")
    # 상태 코드가 200(정상)인지 확인합니다.
    assert response.status_code == 200
    # 반환된 JSON 데이터가 예상과 맞는지 확인합니다.
    assert response.json() == {"message": "Hello, FastAPI World!"}

def test_create_item():
    # POST 요청에 JSON 데이터를 담아 보냅니다.
    response = client.post(
        "/items/",
        json={"name": "테스트 상품", "price": 1000}
    )
    assert response.status_code == 200
    assert response.json()["item_name"] == "테스트 상품"
```

- 실행
    - 터미널에 pytest라고 입력하면 위 코드들이 순식간에 실행되면서 에러가 있는지 없는지 알려줌 (CI/CD 자동화의 핵심)

# 2. 백그라운드 작업 (Background Tasks)

- 회워가입 완료 후 환영 이메일을 보내거나, 이미지 크기를 줄이는 작업 등은 시간이 꽤 걸림
- 사용자가 이 작업이 끝날 때까지 로딩 창만 보고 있게 할 수는 없다.
- FastAPI에서는 이를 **백그라운드(뒷단)** 로 던져버릴 수 있음

```python
from fastapi import FastAPI, BackgroundTasks

app = FastAPI()

# 시간이 오래 걸리는 가상의 작업 함수
def write_notification(email: str, message: str):
    with open("log.txt", mode="a") as email_file:
        content = f"notification for {email}: {message}\n"
        email_file.write(content)

@app.post("/send-notification/{email}")
async def send_notification(email: str, background_tasks: BackgroundTasks):
    # 클라이언트에게 응답을 보내는 것과 별개로, 이 함수를 뒷단에서 실행하라고 예약합니다!
    background_tasks.add_task(write_notification, email, message="회원가입을 환영합니다!")
    
    # 이메일 전송 완료를 기다리지 않고 바로 응답을 반환합니다. (응답 속도 엄청 빠름!)
    return {"message": "백그라운드에서 이메일을 전송 중입니다."}
```

# 3. 컨테이너화 (Docker 활용)

- 내 컴퓨터에서는 잘 되는데 서버에 올리니까 에러 발생
- 이런 상황을 방지하기 위해 앱과 그 앱이 실행되는 환경(파이썬 버전, 라이브러리 등)을 통째로 압축하는 기술이 Docker
- 프로젝트 폴더에 Dockerfile이라는 이름(확장자 없음)으로 파일을 하나 만든다.

```docker
# 1. 파이썬 3.11 환경을 가져옵니다.
FROM python:3.11-slim

# 2. 컨테이너 안의 작업 폴더를 /app으로 지정합니다.
WORKDIR /app

# 3. 필요한 라이브러리 목록 파일을 복사하고 설치합니다.
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 4. 내 프로젝트 코드 전체를 컨테이너 안으로 복사합니다.
COPY . .

# 5. 컨테이너가 켜질 때 실행할 명령어를 지정합니다.
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

# 4. 배포

- 지금까지 개발할 떄 썼던 **uvicorn main:app —reload** 는 개발용이다.
- 실제 운영(Production) 환경에서는 트래픽을 효율적으로 관리하고, 서버가 죽으면 다시 살려주는 매니저가 필요한데,
- 파이썬에서는 Gunicorn이 그 역할을 함
- Gunicorn을 지휘관(Process Manager)으로 세우고, Uvicorn을 그 밑에서 일하는 일꾼(Worker)으로 조합하는 것이 FastAPI의 국률임.

```bash
# Gunicorn과 Uvicorn 워커 클래스를 함께 설치
pip install gunicorn uvicorn

# 워커(일꾼)를 4명 고용해서 프로덕션 모드로 서버 실행!
gunicorn main:app -w 4 -k uvicorn.workers.UvicornWorker --bind 0.0.0.0:8000
```

- **-w 4**: 일꾼(Worker) 프로세스를 4개 띄운다는 뜻, 서버의 CPU코어 수에 맞게 조절
- **-k: Gunicorn이 사용할 일꾼의 종류를 Uvicorn으로 지정하여 비동기 처리 성능을 챙김**
