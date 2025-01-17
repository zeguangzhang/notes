1. 获取学校列表接口（所有的筛选字段见fastapi的docs）
```
curl -X 'GET' \
  'http://0.0.0.0:8000/api/school?pn=1&rn=10&province_name=%E6%B2%B3%E5%8C%97&f985=1' \
  -H 'accept: application/json' \
  -H 'X-USER-ID: 124' \
  -H 'X-USER-NAME: 123'
```
[图片]
返回：
```
{
  "list": [
    {
      "id": 4666,
      "school_id": 1218,
      "create_date": "2022-06-23",
      "name": "东北大学秦皇岛分校",
      "level_name": "普通本科",
      "type_name": "理工类",
      "province_name": "河北",
      "city_name": "秦皇岛市",
      "address": "河北省秦皇岛经济技术开发区泰山路143号",
      "nature_name": "公办",
      "dual_class_name": "双一流",
      "f211": 1,
      "f985": 1,
      "province_id": 13
    }
  ],
  "count": 1
}
```
