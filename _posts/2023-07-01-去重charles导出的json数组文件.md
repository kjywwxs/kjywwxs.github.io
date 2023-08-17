---
categories: 数据处理
tags: 测试 charles python
---

## 前言

当没有任何接口自动化基础设施建设的时候，我们测试一般就是在app上点点点，这时，我们应该如何统计这个测试过程中，涉及的接口覆盖率是多少呢？

思考之后，一个可行的方案是在点点点的过程中进行抓包，在抓包之后对数据进行导出
然后笔者就尝试使用charles对接口进行抓包，然后导出为chjs文件。这时在分析这个json文件时，发现了一些问题：

1. 手动测试过程中难免会有很多重复的接口，干扰我们判断。
2. 一次手动全功能测试下来，涉及的接口非常多，导出的接口文件体积非常大，不仅打开用时很久，也不利于这个文件的复用（比如用于接口测试的回归验证）

为此，衍生出了以下需求：

1. 对导出的json数组文件进行去重
2. 对导出的json数组文件进行分类

## 去重实现

### json数组结构

首先，我们看一下导出的json数组的大致结构：

```json
[
    {
        "status": "COMPLETE",
        "method": "GET",
        "protocolVersion": "HTTP/1.1",
        "scheme": "http",
        "host": "",
        "actualPort": 80,
        "path": "",
        "query": "",
        "request": {
            "sizes": {},
            "mimeType": "application/json",
            "charset": "utf-8",
            "contentEncoding": null,
            "header": {
                "firstLine": "",
                "headers": [
                    {
                        "name": "Connection",
                        "value": "keep-alive"
                    }
                ]
            }
        },
        "response": {
            "status": 200,
            "sizes": {},
            "mimeType": "application/json",
            "charset": "UTF-8",
            "contentEncoding": null,
            "header": {
                "firstLine": "HTTP/1.1 200 ",
                "headers": [
                    {
                        "name": "Server",
                        "value": "nginx/1.10.3 (Ubuntu)"
                    }
                ]
            },
            "body": {
                "text": "",
                "charset": "UTF-8"
            }
        }
    }
]
```

可以看到，json数组中的json是嵌套结构，我们接口测试中区分接口，不单单根据path字段区分用例，还需要根据query,requests.headers,request.body等，所以我们需要构造一个通用的去重方法

### 去重实现初版

```python
def get_dict_value(d:dict, k:str):
    '''
    用于处理字典的嵌套取值
    params k : 类似"request.header.headers"
    return: d[request][header][headers]
    '''
    keys = k.split('.')
    value = d
    
    for key in keys:
        value = value.get(key)
    
    return value


def unique_json_arr_by_fields(data: list[dict], fields:list[str]=None, exclude:list[str]=None) -> list[dict]:
    '''
    给json数组去重,这个方法适合文件不大时使用
    param data : 由json数组转为字典的字典数组
    param fields: 去重的依据字段，只有全部相同才会算作一条记录,支持多重嵌套，如"request.header.headers"
    param exclude: 去重的排除字段，目前只支持第一重字段
    return : 去重后的字典数组
    '''
    # 如果未指定需要去重的字段，则默认以所有字段作为组合字段
    if not fields:
        fields = list(data[0].keys())
    
    # 如果指定了排除字段，则将其从组合字段中删除
    if exclude:
        for field in exclude:
            if field in fields:
                fields.remove(field)
    
    # 使用一个字典对象来记录出现过的组合字段的值
    unique = {}
    
    # 遍历数据，并根据组合字段的值进行去重
    for e in data:
        key = tuple(json.dumps(get_dict_value(e,field)) for field in fields)
        # 使用的字段值合成的字符串作为键
        if key not in unique:
            unique[key] = e
    
    # 返回去重后的结果列表
    return list(unique.values())


def unique_json_arr_file_by_fields(input_file_path, output_file_path, fields:list[str]=None, exclude:list[str]=None):
    '''
    对json数组文件进行去重
    param fields: 去重的依据字段，只有全部相同才会算作一条记录,支持多重嵌套，如"request.header.headers"
    param exclude: 去重的排除字段，目前只支持第一重字段
    return: None
    '''
    with open(input_file_path) as f:
        try:
            tmp = json.load(f)
            print("去重前共{}条".format(len(tmp)))
            result = unique_json_arr_by_fields(tmp,fields=fields,exclude=exclude)
            print("去重后共{}条".format(len(result)))
            # 将字典数组输出为json格式，存储到文件中
            with open(output_file_path, 'w') as f:
                json.dump(result, f)
        except Exception as e:
            print(e)

```

### 去重实现优化

考虑到文件较大，为了能处理巨大的甚至超过内存的数据量，使用ijson库将数据分布加载到内存中，维护一个group字典记录该类别分组有没有没写入过文件，代码如下：

```python
import ijson
import json

def unique_large_json_arr_file_by_fields(input_file_path, output_file_path, fields:list[str]=None, exclude:list[str]=None):
    '''
    对json数组文件进行去重
    param fields: 去重的依据字段，只有全部相同才会算作一条记录,支持多重嵌套，如"request.header.headers"
    param exclude: 去重的排除字段，目前只支持第一重字段
    return: None
    '''
    output_file = open(output_file_path,mode='wb')
    output_file.write('['.encode("utf-8"))
    with open(input_file_path, mode='r', encoding='utf-8') as f:
        # items = ijson.items(f, 'item') # 返回一个生成器对象
        group = {}
        try:
            for item in ijson.items(f, 'item'):
                # 判断根据json指定字段值，，看是否已存在分组
                # 如果item分组已经存在，则跳过，
                # 如果item分组不存在，则新建分组分组，分组的区分标志为group中的键
                # 将新加入的json写入文件
                # 使用的字段值合成的字符串作为键
                key = tuple(json.dumps(get_dict_value(item,field)) for field in fields)
                if key in group:
                    continue
                else:
                    group[key] = True # 标识这一分类已经有一条了
                    output_file.write((json.dumps(item) + ",").encode("utf-8"))
        except Exception as e:
            print(e)
    output_file.seek(-1, 1) # 控制文件指针 -1 表示左偏移1字节 ","在utf8编码下是1字节，1表示总当前位置开始
    output_file.truncate() # 从当前位置向后截断
    output_file.write(']'.encode("utf-8"))
    output_file.close()

```

## 分类实现

暂未实现，后续再做考虑

## 展望

导出的charles文件可以直接转成接口自动化用例的配置文件，定时执行任务进行回归验证，后端悄悄改东西导致线上bug能及时发现。