# 0105 개선사항

1. 센서 제어(sensor_ctrl.py)

- 센서 데이터를 읽고 장치를 직접 제어하는 물리 계층 역할

### 1. 무한 루프의 예외 처리와 자원 해제

-while True 루프 내부에서 예외가 발생하면 프로그램이 완전히 멈출 수 있음.

- 개선 : try...except...finally. 구조를 사용해 오류가 나더라도 GPIO.cleanup()이 확실히 실행되도록 해야함.

<br>

### 2. 센서 읽기 로직의 안정화

- DHT11 센서는 하드웨어 특성상 읽기 오류가 자주 발생
- 개선 : `get_sensor_data` 함수 내의 except 구문에 다시 읽으려 시도하는데 여기서 에러가 나면 프로그램이 죽어버림. None을 반환하거나 이전 값을 유지하는 로직으로 안정화. -> 데이터 분석 시 None값이 더 나을 지 이전 값을 유지하는 것이 좋을지 고민해보았는데 DHT11 센서는 측정 주기가 최소 1초로 1초에 보내는 데이터 양이 적은 편이고 서버로 5~8초에 한 번씩 데이터를 전송하므로 None으로 처리.

<br>

### 3. 중복 코드의 모듈화(DRY 원칙)

- `ctrl_devices`함수 내에서 LED, Fan, Cooler 등의 제어 로직이 거의 동일한 패턴(DB조회 -> 조건 비교 -> GPIO 출력 -> DB 업데이트)으로 반복 되고 있음.
- 개선 : 장치 유형과 조건을 인자로 받는 공통 함수를 만들어 코드 길이를 줄이고 가독성을 높이기

<br>

### 4. 물주기(Water) 로직의 지연 현상

- 현재 토양이 건조하면 `time.sleep(8)`을 사용하여 8초간 대기함. 이 동안 다른 센서 데이터 수집이나 장치 제어가 모두 중단됨.
- 개선 : 비동기(asyncio)를 사용하거나, "시작 시간"만 기록하고 루프를 돌면서 시간이 다 되었는지 체크하는 방식 권장.

<br>

<br>

2. 서버(local_server.py)

- 외부와 통신하며 로컬 DB를 관리하는 게이트웨이 역할

### 1. 데이터베이스 연결 관리

- 현재 `conn = sqlite3.connect(..., check_same_thread=False)`를 전역으로 사용 중. FastAPI는 비동기로 동작하므로 여러 요청이 동시에 들어올 때 SQLite의 데이터 무결성 문제가 생길 수 있음.
- 개선 : 요청마다 DB 세션을 열고 닫거나, `SQLAlchemy` 같은 ORM을 사용하여 연결 풀(Connection Pool)을 관리하는 것이 안전\

### 2. 하드코딩된 설정 분리

- URL, user_id, 센서 임계값 등이 코드 내에 직접 젹혀있음.
- 개선 : `.env` 파일이나 별도의 `config.yaml` 파일을 사용하여 설정 값을 관리. 환경에 따라 UIRL이 바뀔 때 코드 수정을 최소화 할 수 있음.

### 3. API 엔드포인트 설계(RESTful)

- 현재 `/update`,`/level`등의 엔드포인트가 명확한 리소스 중심이 아님.
- 개선 :
  - `POST /devices/{device_id}/control`(장치 제어)
  - `PUT /settings/thresholds`(임계값 수정)

### 4. 카메라 캡처 및 업로드 최적화

- `get_image` 함수 내에서 `requests.post`를 호출할 때 오타(`response.status_code` -> `status_code`). 또한 이미지 업로드는 네트워크 상황에 따라 오래 걸릴 수 있음.
- 개선 : `BackgroundTasks`를 사용하여 이미지 캡처 후 업로드는 백그라운드에서 처리하고, 사용자에게는 즉시 응답을 주는 것이 좋음.

3. 공통 개선 사항 : 아키텍처 및 보안

### 1. 로깅(Logging) 도입

- 모두 `print()`를 사용해 로그를 출력하고 있음.
- 개선 : 파이썬 표준 `logging` 모듈을 사용. 로그 레벨(INFO, ERROR, DEBUG)을 구분할 수 있고, 파일로 저장해 추후 시스템 장애 발생 시 원인 파악이 용이

### 2. 동기화 문제 (Race Condition)

- `sensor_ctrl.py`와 `local_server.py`가 동일한 `farm.dab` 파일을 공유함. 한쪽에서 쓰고 있을 때 다른 쪽에서 접근하면 `database is locked`에러가 발생할 가능성 높음.
- 개선 : WAL(Write-Ahead Logging) 모드 활성화 : ```cursor.execute("PRAGMA journal_mode=WAL"). 또는 DB를 통하지 않고 두 프로세스 간 통신(IPC)을 위해 Redis나 Message Queue를 사용하는 방법도 있음.

cursor.execute("PRAGMA journal_mode=WAL")

# Summer

This is a project template for a project based on FastAPI.

Summer uses several helping modules in summer-toolkit(https://github.com/intotherealworld/summer-toolkit).

- RouterScanner: scan and include APIRouters which comply with the naming rule automatically
- SimpleJinja2Templates: find the template's absolute path with just a template directory name
- Environment: properties and phase management module. It can manage properties each deployment phase separately

## Usage

```
> git clone https://github.com/intotherealworld/summer.git
> cd summer
> pip install -r requirements.txt
> python local_server.py
```

### How to apply your project name

#### 1. change directory name

```
> git clone https://github.com/intotherealworld/summer.git
> mv summer your_project_name
> cd your_project_name
> rm -rf .git
> mv summer your_project_name
```

#### 2. change project name in Dockerfile

```dockerfile
# change "summer" to your project name
ARG SUMMER_PROJECT_NAME="summer"
```

#### 3. apply your project metadata to properties.yml

```yaml
# do not change this key "summer".
# just change the series of "your_" below.
summer:
  project-name: "your_project_name"
  docs:
    title: "your_project_name"
    description: "your_project_description"
    version: "your_project_version"
```

### Docker

```commandline
> docker build -t summer -f ./deploymemt/Dockerfile .
> docker run -p 5000:5000 --name summer summer
```

## Modules (in summer-toolkit)

### RouterScanner

filename pattern:

```
# "router" is cutomizable.
*_router.py
```

coding convention

```
# The part of asterisk(*) must be the same.
*_router = APIRouter(tags=...)
```

example

```
root_router.py

root_router = APIRouter(tags=['root'])
```

### SimpleJinja2Templates

```
# Just use like this, if the directory name for the templates is 'templates'
templates = SimpleJinja2Templates()

# specify the directory name
templates = SimpleJinja2Templates(directory='directory_name')
```

example

> An example is in the [root_router.py](https://github.com/intotherealworld/summer/blob/main/summer/root_router.py)

### Environment

```
env = Environment()
title = env.get_props('summer.docs.title')
```

Setting deployment phase

```
# Environment Variable
SUMMER_DEPLOYMENT_PHASE=your_phase

# The file name of properties must be equal to the value of environment variable.
properties-your_phase.yml
```

If there are no environment variable named 'SUMMER_DEPLOYMENT_PHASE', only the properties.yml is used. The default properties are merged with the phase properties. A phase property which has the same name with a default property overrides it.
