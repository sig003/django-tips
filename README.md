# django-tips

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
