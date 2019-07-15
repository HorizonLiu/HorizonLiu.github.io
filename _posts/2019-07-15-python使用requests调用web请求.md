### python使用requests调用web请求

在项目中，我们经常会需要去测试接口的性能（并发/响应时间等），也会需要通过接口去批量生成测试数据（如批量生成测试账号/账号票据等），这个时候，使用python这样轻量级的脚本工具是很方便的。

本文主要介绍介绍py模拟curl请求的简单使用方法。

#### 接口协议

![image-20190715164718671](http://ww3.sinaimg.cn/large/006tNc79gy1g50mcqubrzj30to0qm77t.jpg)

#### 模拟curl请求

如接口协议所示，请求体中需要一个userList的参数，该参数是一个list。

```python
# 方法一：通过构造json字符串构造请求体
#coding:utf-8
import requests
import time
#import json

# 请求头
headers = {
	"Content-Type": "application/json; charset=UTF-8",
	"companyId": "1"
}

def importUser():
    # 接口
	url = "https://localhost:18081/oauth/resource/user/import?access_token=a07ae252-8f05-4063-84f3-5341a6d10d02"
    # 请求体 -- json字符串
	payload = "{\"userList\":[";
    # 组装list
	for phone in range(20000000000, 20000010000):
		payload = payload + "{\"nation\":86,\"phone\":\"" + str(phone) + "\"}"
		if phone == 20000009999:
			pass
		else:
			payload = payload + ","
	payload = payload + "]}"
	# print(payload)
	currTime = time.time()
	results = requests.post(url, data=payload, headers=headers)
	print(results)
    # 记录请求时长
	print("consume time: ", time.time()- currTime)

if __name__ == '__main__':
	importUser()
```

```python
# 方法二：使用json包构造请求体
#coding:utf-8
import requests
import time
import json

# 请求头
headers = {
	"Content-Type": "application/json; charset=UTF-8",
	"companyId": "1"
}

def importUser():
    # 接口
	url = "https://localhost:18081/oauth/resource/user/import?access_token=a07ae252-8f05-4063-84f3-5341a6d10d02"
    formData = {"nation":"86","phone":"13011111111"}
	# print(json.dumps(formData))
	currTime = time.time()
	results = requests.post(url, data=json.dumps(formData), headers=headers)
	print(results)
    # 记录请求时长
	print("consume time: ", time.time()- currTime)

if __name__ == '__main__':
	importUser()
```

第一种方法可模拟请求体为任意类型的请求，对于请求字段含有list类型的请求，可任意往list中塞数据，而对于第二种方法，目前没有找到好的方式往list中塞任意多的数据。

