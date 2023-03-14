# django-tips

## apscheudler를 이용한 스케쥴러 등록
```
//참조: https://dev.to/brightside/scheduling-tasks-using-apscheduler-in-django-2dbl

//1. 모듈 설치
pip install apscheduler


//2. 스케쥴러를 관리할 앱 생성, 기존 앱에 진행해도 상관없음
python manage.py startapp scheduler


//3. INSTALLED_APPS 에 위에서 생성한 앱 등록
//<DJANGOPROJECT>/setting.py

INSTALLED_APPS = [
		...
    'schedulers',
]


//4. 실행할 함수 등록
//<DJANGOPROJECT>/scheduler/views.py

def Hello():
	print('Hello World')


//5. 실행 방법 등록
//<DJANGOPROJECT>/scheduler/updater.py

from apscheduler.schedulers.background import BackgroundScheduler
from .views import Hello


def start():
    scheduler = BackgroundScheduler()
    scheduler.add_job(Hello, 'interval', seconds=3)
    scheduler.start()


//6. Start
//<DJANGOPROJECT>/scheduler/apps.py

from django.apps import AppConfig


class SchedulersConfig(AppConfig):
    default_auto_field = 'django.db.models.BigAutoField'
    name = 'schedulers'

    def ready(self):
        from . import updater
        updater.start()

```


## Django + Pymongo + Mongodb 사용 시 배열 필드에 Insert, Delete
### Insert
```
//아래와 같은 배열 필드가 있음
{
  "data": ["2021", "2022"]
}
```
```
//"2023" 추가

data = {
    "$push": {
        "data": {
            "$each": ["2023"]
        }
    }
}

collection.update_one({}, data)
```
```
//결과
{
  "data": ["2021", "2022", "2023"]
}

//$each 안 써주면 아래와 같이 배열안에 배열 생성 됨
{
  "data": ["2021", "2022", ["2023"]]
}
```
### Delete
```
data = {
    "$pullAll": {
        "data": {
            ["2023"]
        }
    }
}

collection.update_one({}, data)
```
```
//결과
{
  "data": ["2021", "2022"]
}
```

## Pymongo의 결과물을 for문 돌면서 key - value 할당해 json 리턴
```
import pymongo
import json

# MongoDB와 연결
client = pymongo.MongoClient("mongodb://localhost:27017/")
db = client["mydatabase"]
collection = db["mycollection"]

# MongoDB에서 데이터 가져오기
results = collection.find()

# Python 리스트에 데이터 저장
data = []
for result in results:
    data.append(result)

# Python 객체에 새로운 key-value 쌍 추가
for d in data:
    d["new_key"] = "new_value"

# Python 객체를 json으로 직렬화
json_results = json.dumps(data)

# 결과 출력
print(json_results)
```

## Mongodb에서 embedded document 업데이트 할 때 dot notation을 이용한 부분 업데이트 처리
```
//mongodb 스키마에 아래와 같은 embedded document 가 있다고 가정

payment {
 success: null,
 error: null,
}

//쿼리 조건문
condition = {
 'key': 'abcd'
}

//아래과 같이 업데이트 해 버리면 정의하지 않은 다른 컬럼은 없어짐 (error 컬럼 날아감)
//아래에 정의한 컬럼으로 바꿔치기 해 버림
data = {
 '$set': {
  'payment': {
   'success': 'ok'
  }
 }
}

//아래와 같이 하면 정의한 컬럼만 업데이트 침
//dot notation
data = {
 '$set': {
  'payment.success': 'ok'
 }
}

db.update_one(condition, data)
```
