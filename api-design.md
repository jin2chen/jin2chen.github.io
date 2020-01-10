[TOC]

# API 设计规范 Version 1.0

## HTTP状态码规范
* 系统框架返回`400`, `401`, `403`, `404`, `500`状态码
* 数据验证错误返回`422`状态码
* 业务逻辑错误等需要通知给客户端，均返回`400`状态码
* 服务器端未知错误，需返回`500`状态码
* 成功均返回`200`状态码
* 批量操作部分成功返回`229`状态码

## Response 错误信息数据结构
### <a name="errorResponseObject"></a>Error Response Object

Field Name |                       Type                        | Description
-----------|:-------------------------------------------------:|-----------------
code       |                      integer                      | 错误代码
message    |                      string                       | 错误描述
fields     | [[Validated Error Object](#validatedErrorObject)] | 当有数据验证错误时，此字段会出现

### <a name="validatedErrorObject"></a>Validated Error Object
Field Name |  Type  | Description
-----------|:------:|------------
filed      | string | 字段名
message    | string | 错误描述

```js
{
  "code": 1001,
  "message": "Token 不正确"
}
```

```js
{
  "code": 2000,
  "message": "数据验证错误",
  "fields": [{
    "field": "name",
    "message": "超过20个字符"
  }]
}
```

## Response 分页数据结构
Filed Name |                  Type                  | Description
-----------|:--------------------------------------:|------------
items      |                [Object]                | 数据
\_links    | [Pagination Link Object](#linksObject) | 页码链接
\_meta     | [Pagination Meta Object](#metaObject)  | 分页数据

### <a name="metaObject"></a>Pagination Meta Object
Filed Name  |  Type   | Description
------------|:-------:|------------
totalCount  | integer | 总记录数
pageCount   | integer | 总分页数
currentPage | integer | 当前页码
perPage     | integer | 每页记录数

### <a name="linksObject"></a>Pagination Links Object
Filed Name |            Type            | Description
-----------|:--------------------------:|------------
self       | [Link Object](#linkObject) | 当前页
first      | [Link Object](#linkObject) | 第一页
prev       | [Link Object](#linkObject) | 前一页
next       | [Link Object](#linkObject) | 后一页
last       | [Link Object](#linkObject) | 最后页

### <a name="linkObject"></a> Pagination Link Object
Filed Name |  Type  | Description
-----------|:------:|------------
href       | string | 链接地址

```js
{
  "items": [
    {
      "id": "2",
      "email": "windy.luo@starlight-sms.com",
      "name": "马丁罗",
      "status": "ACTIVE"
    }
  ],
  "_links": {
    "self": {
      "href": "http://127.0.0.1:8810/v1/users?per-page=1&page=2"
    },
    "first": {
      "href": "http://127.0.0.1:8810/v1/users?per-page=1&page=1"
    },
    "prev": {
      "href": "http://127.0.0.1:8810/v1/users?per-page=1&page=1"
    },
    "next": {
      "href": "http://127.0.0.1:8810/v1/users?per-page=1&page=3"
    },
    "last": {
      "href": "http://127.0.0.1:8810/v1/users?per-page=1&page=3"
    }
  },
  "_meta": {
    "totalCount": 3,
    "pageCount": 3,
    "currentPage": 2,
    "perPage": 1
  }
}
```

## Resonse Resulst Object
Filed Name  |                      Type                      | Description
------------|:----------------------------------------------:|-------------------------------
code        |                    integer                     | `0` 表示成功，`3` 表示部分成功`如果是批量操作`
message     |                     string                     | `SUCCESS` or `PARTIAL_SUCCESS`
result      |                     Object                     | 如果需要返回数据将放在此字段里
failedItems | [[Bulk Item Failure Object](#bulkItemFailure)] | 描述在批量操作中，操作失败的记录

### <a name="bulkItemFailure"></a> Bulk Item Failure Object
Filed Name |             Type             | Description
-----------|:----------------------------:|-----------------
index      |           integer            | 失败记录在批量请求数组中的索引值
error      | [Error Object](#errorObject) | 失败错误信息

### <a name="errorObject"></a> Error Object
Filed Name |  Type   | Description
-----------|:-------:|------------
code       | integer | 错误代码
message    | string  | 错误信息

```js
{
  "code": 0,
  "message": "SUCCESS"
}
```

```js
{
  "code": 3,
  "message": "PARTIAL_SUCCESS",
  "failedItems": [
    {
      "index": 3,
      "error": {
        "code": 1024,
        "message": "此记录正在被引用，不能删除"
      }
    }
  ]
}
```

## Response (PUT/POST/GET) 直接返回 Object
```js
{
  "key": "value"
}
```

## 基于Yii2 代码实现

错误抛出不限于 Controller, 通常会在 Model 中抛出业务逻辑与数据验证错误

```php
<?php
namespace rest\versions\v1\controllers;

use yii\base\Model;
use yii\data\ArrayDataProvider;
use rest\libraries\errors\ErrorCode;
use rest\libraries\errors\ValidateException;
use rest\libraries\Controller;
use rest\libraries\errors\ClientException;
use rest\libraries\successes\Success;

class TestController extends Controller
{
    /**
     * 分页列表
     *
     * ```
     * {
     *   "items": [
     *     {
     *       "id": 1,
     *       "name": "mole"
     *     }
     *   ],
     *   "_links": {
     *     "self": {
     *       "href": "http://127.0.0.1:8810/v1/users?page=1"
     *     }
     *   },
     *   "_meta": {
     *     "totalCount": 3,
     *     "pageCount": 1,
     *     "currentPage": 1,
     *     "perPage": 20
     *   }
     * }
     * ```
     *
     * @return ArrayDataProvider
     */
    public function actionIndex()
    {
        return new ArrayDataProvider([
            'allModels' => [],
            'key' => 'id'
        ]);
    }

    /**
     * 返回Object
     *
     * ```
     * {
     *   "id": 1,
     *   "name": "mole"
     * }
     * ```
     *
     * @return Model
     */
    public function actionView()
    {
        return new Model();
    }

    /**
     * 成功
     *
     * ```
     * {
     *   "code": 0,
     *   "message": "SUCCESS"
     * }
     * ```
     *
     * @return Success
     */
    public function actionDelete()
    {
        return new Success();
    }

    /**
     * 批处理部分失败
     *
     * ```
     * {
     *   "code": 3,
     *   "message": "PARTIAL_SUCCESS",
     *   "failedItems": [
     *     {
     *       "index": 3,
     *       "error": {
     *         "code": 3014,
     *         "message": "cant remove."
     *       }
     *     }
     *   ]
     * }
     * ```
     *
     * @return Success
     */
    public function actionPartial()
    {
        return new Success([
            'failedItems' => [
                [
                    'index' => 3,
                    'error' => [
                        'code' => 3124,
                        'message' => 'Cant remove.'
                    ]
                ]
            ]
        ]);
    }

    /**
     * 直接抛出错误信息
     *
     * ```
     * {
     *   "code": 1001
     *   "message": "Token invalid."
     * }
     * ```
     *
     * @throws ClientException
     */
    public function actionError()
    {
        throw new ClientException(ErrorCode::E_DATA_EXIST);
    }

    /**
     * 直接抛出验证错误
     *
     * ```
     * {
     *   "code": 2000,
     *   "message": "数据验证错误",
     *   "fields": [
     *     {
     *       "field": "name",
     *       "message": "超过20个字符"
     *     }
     *   ]
     * }
     * ```
     *
     * @throws ValidateException
     */
    public function actionValidate()
    {
        $model = new Model();
        throw new ValidateException($model);
    }
}
```

## 参考
- http://www.vinaysahni.com/best-practices-for-a-pragmatic-restful-api
- http://baike.baidu.com/view/1790469.htm
