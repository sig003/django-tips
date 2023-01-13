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
