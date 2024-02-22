# django-tips

## Put, Delete Method 처리 방법두 가지
### decorator 로 처리하는 방법
- csrf 토큰 관련 오류가 나서 csrf_exempt 처리해 줘야 함
```
# urls.py
urlpatterns = [
    path('test/', TestView.as_view()),
]
```
```
# views.py
from rest_framework.views import APIView
from django.http import HttpResponse
from rest_framework.decorators import action
from django.views.decorators.csrf import csrf_exempt
from django.utils.decorators import method_decorator

@method_decorator(csrf_exempt, name = 'dispatch')
class TestLogicView(APIView):
     @action(detail=True, methods=['delete'])
         def delete(self, request, slug):
            result = 'ok'
            return HttpResponse(result, status=200)
    
     @action(detail=True, methods=['put'])
         def put(self, request, slug):
            result = 'ok'
            return HttpResponse(result, status=200)
```

### APIView에 pk/slug 담아서 처리하는 방법
- Django의 APIView에서 pub, delete 메서드 이용할 때 해당 리소스의 고유 식별자(pk 또는 slug)를 보내야 함
```
# urls.py
urlpatterns = [
    path('test/<str:slug>', TestView.as_view()),
]
```
```
# views.py
from rest_framework.views import APIView
from django.http import HttpResponse

class TestLogicView(APIView):
    def delete(self, request, slug):
        result = 'ok'
        return HttpResponse(result, status=200)

    def put(self, request, slug):
        result = 'ok'
        return HttpResponse(result, status=200)
``` 

## ALLOWED_HOSTS의 의미
### 정의
- Django의 settings.py에 ALLOWED_HOSTS 를 등록할 수 있음
- 어디서 오는지 막는 기능이 아님 (referer 체크와 다름)
- django가 실행될 수 있는 서버의 도메인이나 ip를 지정하는 것임
  
### HTTP Host Header Attack 에 대한 방어 기능 수행
```
1) Django 가 www.api.com 이라는 도메인에서 동작
2) 해커가 http header의 host 값을 악의적인 사이트로 변조함 ⇒ www.hacker.com
3) 어떤 동작을 수행할 때 www.hacker.com으로 넘김
4) ALLOWED_HOSTS에 www.api.com 만 등록되어 있으면 다른 도메인으로의 동작을 차단할 수 있음
```

### 등록 예제
```
# frontend: front.server.com
# backend (django): www.api.com
# front.server.com을 등록하는 것이 아님
# django가 실행되는 도메인을 기록해야 함

# settings.py
ALLOWED_HOSTS = ['www.api.com']
```

## Django + Mongodb 기반의 BackEnd 환경에서 테스트 DB 구축
### 조건   
- 운영 DB 이외에 테스트 DB가 필요함
- 두 DB 간의 스위칭은 자동으로 이루어져야 함 (실수 방지 목적)
- 운영 서버에서는 운영 DB에 접속하고 테스트 서버에서는 테스트 DB에 접속해야 함

### 환경
- BackEnd: Elasticbeans Talk + Django
- DB: Mongodb Atlas

### 방법
- Mongodb Atlas에서 테스트 DB용 프로젝트 생성
- 이는 운영 DB와 테스트 DB 간에 상호 영향없게 하기 위함
- 테스트 DB는 Mongodb Atlas의 무료 모드로 사용
- 운영 DB의 콜렉션과 데이터를 export 해서 테스트 DB에 import

### settings.py
- Django 의 settings.py 변경
- Elasticbeans talk에 반영된 운영 서버는 AWS_EXECUTION_ENV 값을 가지고 있고 이 값이 있으면 운영 DB 바라봄
- 이외에는 테스트 DB 바라봄
```
# settings.py

import os

if os.environ.get('AWS_EXECUTION_ENV'):
    selected_host = MONGODB_HOST_FILE
else:
    selected_host = MONGODB_TEST_HOST_FILE

DATABASES = {
    'default': {
        'ENGINE': 'djongo',
        'NAME': 'db',
        'ENFORCE_SCHEMA': False,
        'CLIENT': {
            'host': selected_host,
        },
    }
}
```

## Django 패스워드(PBKDF2PasswordHasher) 생성 로직
### 패스워드 생성
```
1. 패스워드 입력
password:abcd


2. 알고리즘으로 hash 생성
password: abcd
salt(random value): pB7aJgypDGpGR837yNcGmb
iteration: 100000
digest: sha256

결과 => hash: 8mDMMPX3i1H2gYb5Emo5vDfF6PKPj18/3YtoDFN6afE=


3. 생성한 hash와 알고리즘을 조합해 최종 패스워드 생성
algorithm: pbkdf2_sha256
iteration: 100000
salt: pB7aJgypDGpGR837yNcGmb
hash: 8mDMMPX3i1H2gYb5Emo5vDfF6PKPj18/3YtoDFN6afE=

결과 => 최종 패스워드: pbkdf2_sha256$100000$pB7aJgypDGpGR837yNcGmb$8mDMMPX3i1H2gYb5Emo5vDfF6PKPj18/3YtoDFN6afE=
```

### 패스워드 비교
```
1. 기존 패스워드에서 알고리즘 추출
기존 패스워드: pbkdf2_sha256$100000$pB7aJgypDGpGR837yNcGmb$8mDMMPX3i1H2gYb5Emo5vDfF6PKPj18/3YtoDFN6afE=

결과 => 
algorithm: pbkdf2_sha256
iteration: 100000
salt: pB7aJgypDGpGR837yNcGmb


2. 비교할 패스워드 입력
password: abcd


3. 기존 패스워드에서 추출한 알고리즘으로 입력한 패스워드의 hash 생성
password: abcd
salt(random value): pB7aJgypDGpGR837yNcGmb
iteration: 100000
digest: sha256

결과 => hash: 8mDMMPX3i1H2gYb5Emo5vDfF6PKPj18/3YtoDFN6afE=


4. 생성한 hash와 알고리즘을 조합해 최종 패스워드 생성
algorithm: pbkdf2_sha256
iteration: 100000
salt: pB7aJgypDGpGR837yNcGmb
hash: 8mDMMPX3i1H2gYb5Emo5vDfF6PKPj18/3YtoDFN6afE=

결과 => 최종 패스워드: pbkdf2_sha256$100000$pB7aJgypDGpGR837yNcGmb$8mDMMPX3i1H2gYb5Emo5vDfF6PKPj18/3YtoDFN6afE=


5. 기존 패스워드와 입력한 패스워드의 최종 패스워드 값으로 비교 실행
기존 패스워드: pbkdf2_sha256$100000$pB7aJgypDGpGR837yNcGmb$8mDMMPX3i1H2gYb5Emo5vDfF6PKPj18/3YtoDFN6afE=

==

입력 패스워드: pbkdf2_sha256$100000$pB7aJgypDGpGR837yNcGmb$8mDMMPX3i1H2gYb5Emo5vDfF6PKPj18/3YtoDFN6afE=
```

### 설명
- $를 기준으로 값을 구분함
- hash 값은 암호화 된 값을 base64 인코딩한 값
- iterations 은 알고리즘을 반복하는 횟수로 django 버전에 따라 다를 수 있음
- 패스워드 값은 hash 값이라 복호화를 할 수 없음
- plaintext로 hash 값을 생성하면 항상 동일한 결과값을 얻을 수 있지만 salt 값 때문에 동일한 결과값을 얻을 수 없음
- salt 값은 랜덤하게 바뀌는 값이므로 같은 plaintext 값을 넣어도 똑같은 암호화 값을 얻을 수 없음
- 암호화 된 값을 복호화 할 수 없으므로 동일한 알고리즘으로 plaintext 값을 암호화하고 이 값으로 기존에 생성된 암호화 값과 비교해야 함
- 동일한 알고리즘으로 재암호 할 수 있게 하려고 algorithm, iterations, salt 값을 패스워드 자체에 붙여 놓는 것임
- plaintext로 동일한 암호화 값이 얻어지지 않아서 문제라고 생각하면 안 됨
- 그렇게 만드려면 salt 값을 고정시키면 되지만 항상 동일한 hash 값이 만들어지므로 보안에 취약해짐
- salt는 동일한 값이 안 나오게 하기 위한 장치임
- 패스워드 탈취를 위해 hash 테이블을 가지고 있으면 모든 hash에 대한 값을 유추할 수 있으므로 보안에 취약해져 salt 같은 값으로 변형을 주는 것임

## 고차 함수로 try/except 공통화해서 코드 간결화하기
```
from utils.py import makeTryExcept 

class SomeClass(APIView):

 def post(self, request):
        return makeTryExcept(lambda: self.mainLogic(request))

    def mainLogic(self, request):
      ...
      # Main Logic
      ...


# utils.py

def makeTryExcept(mainLogicFunction):
    try:
        return mainLogicFunction()
    except Exception as e:
        return HttpResponse(
                      json.dumps({'status': 500, 'result': str(e)},
                      default=json_util.default,                
                      ensure_ascii=False),
                      status=500
                  )
```

## Django-Oauth-Toolkit Custom Overriding
```
# MyApp/urls.py
from .customtoken import CustomTokenView

urlpatterns = [
    path('o/', include('oauth2_provider.urls', namespace='oauth2_provider')),
    path('o/customtoken/', CustomTokenView.as_view(), name='oauth2_provider_custom_token'), # 커스텀 url 추가
    ...
]


# MyApp/customtoken.py
# 아래 파일을 생성
# Django-Oauth-Toolkit의 token 생서 로직 복사
# 원하는 위치에 커스텀 로직 삽입해 기존 로직 overriding 처리

from django.http import HttpResponse
from django.utils.decorators import method_decorator
from django.views.decorators.debug import sensitive_post_parameters
from oauth2_provider.models import get_access_token_model
from oauth2_provider.signals import app_authorized
import json
from django.views.decorators.csrf import csrf_exempt
from oauth2_provider.views.base import TokenView

@method_decorator(csrf_exempt, name="dispatch")
class CustomTokenView(TokenView):
    """
    Implements an endpoint to provide access tokens

    The endpoint is used in the following flows:
    * Authorization code
    * Password
    * Client credentials
    """

    @method_decorator(sensitive_post_parameters("password"))
    def post(self, request, *args, **kwargs):
        url, headers, body, status = self.create_token_response(request)
        if status == 200:
            access_token = json.loads(body).get("access_token")
            if access_token is not None:
                token = get_access_token_model().objects.get(token=access_token)
                app_authorized.send(sender=self, request=request, token=token)
        response = HttpResponse(content=body, status=status)
        
        for k, v in headers.items():
            response[k] = v
        return response


# http request
curl -X POST -d "grant_type=password&username=<user_name>&password=<password>" -u"<client_id>:<client_secret>" http://localhost:8000/o/customtoken/

```

## Django-Oauth-Toolkit access_token, refresh_token 생성 순서
### 1) 최초 토큰 발급
```
oauth2_provider_accesstoken

# 생성
id: 100
source_refresh_token: null
token: aaaaa
```
```
oauth2_provider_refreshtoken

# 생성
id: 200
access_token_id: 100
token: ttttt
revoked: null
```

### 2) access_token 만료 후 refresh_token으로 1회 갱신
```
oauth2_provider_accesstoken

# 생성
id: 101
source_refresh_token: 200
token: bbbbb

# 삭제
id: 100
source_refresh_token: null
token: aaaaa
```
```
oauth2_provider_refreshtoken

# 생성
id: 201
access_token_id: 101
token: uuuuu
revoked: null


# 취소로 업데이트
id: 200
access_token_id: null
token: ttttt
revoked: 2023-04-01T00:00:000
```

### 3) access_token 만료 후 refresh_token으로 2회 갱신
```
oauth2_provider_accesstoken

# 생성
id: 102
source_refresh_token: 201
token: ccccc

# 삭제
id: 101
source_refresh_token: 200
token: bbbbb
```
```
oauth2_provider_refreshtoken

# 생성
id: 202
access_token_id: 102
token: vvvvv
revoked: null

# 취소로 업데이트
id: 201
access_token_id: null
token: uuuuu
revoked: 2023-04-01T00:00:000

# 기존에 취소로 업데이트되어 있음
id: 200
access_token_id: null
token: ttttt
revoked: 2023-04-01T00:00:000
```

## API Http Response 공통 함수화
### 기존 소스

```
#Test/app/views.py

from rest_framework.views import APIView
from rest_framework import status
from bson import json_util
import json
from django.http import HttpResponse

class SomeView(APIView):
  def get(self, request):  
    try:
        serializer = SomeSerializer()
        if serializer.is_valid():
    
          #logic
          ...

           return HttpResponse(json.dumps({'status': 'S', 'data': result}, default=json_util.default, ensure_ascii=False), status=status.HTTP_200_OK)
         return HttpResponse(json.dumps({'status': 'F', 'data': 'Bad Request'}, default=json_util.default, ensure_ascii=False), status=status.HTTP_400_BAD_REQUEST)
    except:
          return HttpResponse(json.dumps({'status': 'F'}, default=json_util.default, ensure_ascii=False), status=status.HTTP_500_INTERNAL_SERVER_ERROR)
```

### 변경 소스

```
#Test/utils.py

from rest_framework import status
from bson import json_util
import json
from django.http import HttpResponse

def generateHttpResponse(statusCode, data=None):
    if statusCode == 200:
        statusResult = 'S'
    elif statusCode == 201:
        statusResult = 'S'
    elif statusCode == 400:
        statusResult = 'F'
        data = 'Bad Request'
    else:
        statusResult = 'F'
        data = 'Internal Server Error'

    return HttpResponse(
            json.dumps(
                {
                    'status': statusResult,
                    'data': data
                },
                default=json_util.default,
                ensure_ascii=False),
            status=statusCode


#Test/app/views.py

from rest_framework.views import APIView
from Test.utils import generateHttpResponse

class SomeView(APIView):
  def get(self, request):  
    try:
        serializer = SomeSerializer()
        if serializer.is_valid():
    
          #logic
          ...

          return generateHttpResponse(200, result)
        return generateHttpResponse(400)
    except:
        return generateHttpResponse(500)
```
   
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
