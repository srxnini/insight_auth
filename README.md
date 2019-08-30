# Insight Auth 服务使用手册

## 目录

- [概述](#概述)
- [Token接口](#Token接口)
  - [获取Code](#获取Code)
  - [获取Token](#获取Token)
  - [微信授权码获取Token](#微信授权码获取Token)
  - [微信UnionId获取Token](#微信UnionId获取Token)
  - [验证Token](#验证Token)
  - [刷新Token](#刷新Token)
  - [注销Token](#注销Token)
- [应用权限接口](#应用权限接口)
  - [获取模块导航](#获取模块导航)
  - [获取模块功能](#获取模块功能)
- [DTO类型说明](#DTO类型说明)

## 概述

Auth 服务是一个依赖于用户数据的、基于Token的用户身份认证服务。每一个Token都必须绑定一个AppId，该Id需要在获取Token时作为参数传入。根据应用的相应设置，系统支持两种Token发放模式：

1. 通用模式，可对不同的设备发放多个相同AppId的的Token，用户可同时在不同的设备登录同一个应用；
2. 专用模式，对不同设备发放相同AppId的时候，之前发放的Token自动失效，用户只能在一个设备上登录同一个应用。

### 主要功能

1. Token的发放(账号/密码|手机号/验证码|微信授权)、验证、刷新、注销；
2. 获取应用的功能模块导航信息，以及指定模块的功能和授权信息；
3. 各类短信验证码的生成-发送、校验、验证；
4. 支付密码的设置/更改、验证。

### 通讯方式

**Insight** 所有的服务都支持 **HTTP/HTTPS** 协议的访问请求，以 **URL Params** 、 **Path Variable** 或 **BODY** 传递参数。如使用 **BODY** 传参，则需使用 **JSON** 格式的请求参数。接口 **URL** 区分大小写，请求以及返回都使用 **UTF-8** 字符集进行编码，接口返回的数据封装为统一的 **JSON** 格式，详见：[**Reply**](#Reply) 数据类型。除获取Token等公开接口外，都需要在请求头的 **Authorization** 字段承载 **AccessToken** 数据。HTTP请求头参数设置如下：

|参数名|参数值|
| ------------ | ------------ |
|Accept|application/json|
|Authorization|AccessToken(Base64编码的字符串)|
|Content-Type|application/json|

## Token接口

### 获取Code

用户可通过此接口获取一个有效时间30秒的32位随机字符串用于生成用户签名。此接口被调用时如用户数据未缓存，则缓存用户数据。同时会在缓存中保存两条过期时间为30秒的String记录。此接口的限流策略为：同一设备的调用间隔需3秒以上，每天调用上限200次。

1. Key:Sign,Value:Code。Sign的算法为 **MD5(MD5(account + password) + Code)**，调用获取Token接口时使用签名验证用户的账号/密码，或手机号/验证码是否正确。如使用手机号/验证码方式登录，则Sign的算法为 **MD5(MD5(mobile + MD5(smsCode)) + Code)**。
2. Key: Code, Value: UserId。调用获取Token接口时可凭签名得到Code，再通过Code得到用户ID。

>注：**password** 为明文密码的 **MD5** 值，数据库中以 **RSA** 算法加密该 **MD5** 值后存储。

请求方法：**GET**

接口URL：**/base/auth/v1.0/tokens/codes**

请求参数如下：

|类型|属性|是否必需|属性说明|
| ------------ | ------------ | ------------ | ------------ |
|String|account|是|登录账号/手机号/邮箱|
|Integer|type|否|登录类型:0.密码登录;1.验证码登录,默认为0|

接口返回数据类型：

|类型|属性|属性说明|
| ------------ | ------------ | ------------ |
|String||Code,30秒内使用有效|

请求参数示例：

```java
@Test
public void testHttpCall() throws IOException {
    // given
    HttpGet request = new HttpGet("http://127.0.0.1:6200/base/auth/v1.0/tokens/codes?account=admin");
    request.add("Accept", "application/json");

    // when
    HttpResponse response = HttpClientBuilder.create().build().execute(request);

    // then
    HttpEntity entity = response.getEntity();
    String jsonString = EntityUtils.toString(entity);
    loger.info(jsonString);
}
```

返回结果示例：

```json
{
  "success": true,
  "code": 200,
  "message": "请求成功",
  "data": "404a257bc35a4540aed079dc4b48d957",
  "option": null
}
```

[回目录](#目录)

### 获取Token

用户可调用此接口获取访问令牌、刷新令牌、令牌过期时间、令牌失效时间和用户信息。signature的计算方法是 **MD5(MD5(account\|mobile\|email + MD5(password\|smsCode)) + Code)** 。此接口的限流策略为：同一设备的调用间隔需3秒以上，每天调用上限200次。

请求方法：**POST**

接口URL：**/base/auth/v1.0/tokens**

请求参数如下：

|类型|属性|是否必需|属性说明|
| ------------ | ------------ | ------------ | ------------ |
|String|appId|是|应用ID|
|String|tenantId|否|租户ID|
|String|deptId|否|登录部门ID|
|String|account|是|登录账号/手机号/邮箱|
|String|signature|是|签名:MD5(MD5(account\|mobile + MD5(password\|smsCode)) + Code)|
|String|deviceId|否|用户设备ID|

接口返回数据类型：

|类型|属性|属性说明|
| ------------ | ------------ | ------------ |
|String|accessToken|访问用令牌|
|String|refreshToken|刷新用令牌|
|Integer|expire|令牌过期时间(毫秒)|
|Integer|failure|令牌失效时间(毫秒)|
|[UserInfo](#UserInfo)|userInfo|用户信息|

请求参数示例：

```json
{
  "account": "admin",
  "signature": "9c110b1fcd11a35e495aabbd7215eb3c",
  "appId": "9dd99dd9e6df467a8207d05ea5581125",
  "tenantId": "2564cd559cd340f0b81409723fd8632a"
}
```

返回结果示例：

```json
{
  "success": true,
  "code": 200,
  "message": "请求成功",
  "data": {
    "accessToken": "eyJpZCI6IjQwNGEyNTdiYzM1YTQ1NDBhZWQwNzlkYzRiNDhkOTU3IiwidXNlcklkIjoiMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAiLCJ1c2VyTmFtZSI6bnVsbCwic2VjcmV0IjoiNzE5ODQ2MjU0NTY3NGNmY2I4MzRjZTkxYThjZTI0NGYifQ==",
    "refreshToken": "eyJpZCI6IjQwNGEyNTdiYzM1YTQ1NDBhZWQwNzlkYzRiNDhkOTU3IiwidXNlcklkIjoiMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAiLCJ1c2VyTmFtZSI6bnVsbCwic2VjcmV0IjoiYjRhOWNhNDE2ODFlNGUyNjg4ZTU3NjI4ODdmZDE4MjEifQ==",
    "expire": 7200000,
    "failure": 86400000,
    "userInfo": {
      "id": "00000000000000000000000000000000",
      "tenantId": null,
      "deptId": null,
      "code": null,
      "name": "系统管理员",
      "account": "admin",
      "mobile": null,
      "email": null,
      "headImg": "http://image.insight.com/cloudfile/headimgs/default.png",
      "builtin": true,
      "createdTime": "2019-05-17 05:28:33"
    }
  },
  "option": null
}
```

[回目录](#目录)

### 微信授权码获取Token

此接口支持通过微信授权获取访问令牌、刷新令牌、令牌过期时间、令牌失效时间和用户信息。如该微信号未绑定用户，则缓存该微信号的 [微信用户信息](#WeChatUser) 30分钟并返回该数据。前端应用可使用微信用户信息中的UnionId调用 [微信UnionId获取Token](#微信UnionId获取Token) 接口，将该UnionId绑定到指定手机号的用户，并获取Token。

请求方法：**POST**

接口URL：**/base/auth/v1.0/tokens/withWechatCode**

请求参数如下：

|类型|属性|是否必需|属性说明|
| ------------ | ------------ | ------------ | ------------ |
|String|appId|是|应用ID|
|String|tenantId|否|租户ID|
|String|deptId|否|登录部门ID|
|String|weChatAppId|是|微信appId|
|String|code|是|微信授权码|
|String|deviceId|否|用户设备ID|

接口返回数据类型：

|类型|属性|属性说明|
| ------------ | ------------ | ------------ |
|String|accessToken|访问用令牌|
|String|refreshToken|刷新用令牌|
|Integer|expire|令牌过期时间(毫秒)|
|Integer|failure|令牌失效时间(毫秒)|
|[UserInfo](#UserInfo)|userInfo|用户信息|

请求参数示例：

```json
{
  "code": "6IjQwNGEyNTdiYzM1YTQ1NDBhZWQw",
  "weChatAppId": "6821a080527d775101b624d632899fee",
  "appId": "9dd99dd9e6df467a8207d05ea5581125"
}
```

正常返回结果示例：

```json
{
  "success": true,
  "code": 200,
  "message": "请求成功",
  "data": {
    "accessToken": "eyJpZCI6IjQwNGEyNTdiYzM1YTQ1NDBhZWQwNzlkYzRiNDhkOTU3IiwidXNlcklkIjoiMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAiLCJ1c2VyTmFtZSI6bnVsbCwic2VjcmV0IjoiNzE5ODQ2MjU0NTY3NGNmY2I4MzRjZTkxYThjZTI0NGYifQ==",
    "refreshToken": "eyJpZCI6IjQwNGEyNTdiYzM1YTQ1NDBhZWQwNzlkYzRiNDhkOTU3IiwidXNlcklkIjoiMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAiLCJ1c2VyTmFtZSI6bnVsbCwic2VjcmV0IjoiYjRhOWNhNDE2ODFlNGUyNjg4ZTU3NjI4ODdmZDE4MjEifQ==",
    "expire": 7200000,
    "failure": 86400000,
    "userInfo": {
      "id": "00000000000000000000000000000000",
      "tenantId": null,
      "deptId": null,
      "code": null,
      "name": "系统管理员",
      "account": "admin",
      "mobile": null,
      "email": null,
      "headImg": "http://image.insight.com/cloudfile/headimgs/default.png",
      "builtin": true,
      "createdTime": "2019-05-17 05:28:33"
    }
  },
  "option": null
}
```

用户不存在时返回结果示例：

```json
{
  "success": true,
  "code": 200,
  "message": "请求成功",
  "data": {
    "unionid": "o6_bmasdasdsad6_2sgVt7hMZOPfL",
    "openid": "o6_bmjrPTlm6_2sgVt7hMZOPfL2M",
    "nickname": "达明",
    "sex": 1,
    "country": "中国",
    "province": "上海",
    "city": "上海",
    "headimgurl": "http://thirdwx.qlogo.cn/mmopen/xbIQx1GRqdvyqkMMhEaGOX802l1CyqMJNgUzK/0",
    "language": "zh_CN"
  },
  "option": false
}
```

[回目录](#目录)

### 微信UnionId获取Token

此接口支持通过微信UnionId获取访问令牌、刷新令牌、令牌过期时间、令牌失效时间和用户信息。同时将用户提供的UnionId绑定到指定手机号的用户。绑定过程如下：

1. 根据 UnionId 和 weChatAppId 在缓存中找到微信用户信息，如未找到缓存数据则返回参数错误；
2. 根据手机号查询用户，如用户不存在，则根据微信用户信息创建用户并返回Token；
3. 如允许更新绑定微信号则绑定新的 UnionId 后返回Token；
4. 如不允许更新绑定微信号则返回用户已绑定其他微信号的错误；

请求方法：**POST**

接口URL：**/base/auth/v1.0/tokens/withWechatUnionId**

请求参数如下：

|类型|属性|是否必需|属性说明|
| ------------ | ------------ | ------------ | ------------ |
|String|appId|是|应用ID|
|String|tenantId|否|租户ID|
|String|deptId|否|登录部门ID|
|String|weChatAppId|是|微信appId|
|String|unionId|是|微信用户唯一ID|
|String|isReplace|否|是否更新微信号|
|String|deviceId|否|用户设备ID|

接口返回数据类型：

|类型|属性|属性说明|
| ------------ | ------------ | ------------ |
|String|accessToken|访问用令牌|
|String|refreshToken|刷新用令牌|
|Integer|expire|令牌过期时间(毫秒)|
|Integer|failure|令牌失效时间(毫秒)|
|[UserInfo](#UserInfo)|userInfo|用户信息|

请求参数示例：

```json
{
  "unionId": "6IjQwNGEyNTdiYzM1YTQ1NDBhZWQw",
  "weChatAppId": "6821a080527d775101b624d632899fee",
  "appId": "9dd99dd9e6df467a8207d05ea5581125"
}
```

返回结果示例：

```json
{
  "success": true,
  "code": 200,
  "message": "请求成功",
  "data": {
    "accessToken": "eyJpZCI6IjQwNGEyNTdiYzM1YTQ1NDBhZWQwNzlkYzRiNDhkOTU3IiwidXNlcklkIjoiMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAiLCJ1c2VyTmFtZSI6bnVsbCwic2VjcmV0IjoiNzE5ODQ2MjU0NTY3NGNmY2I4MzRjZTkxYThjZTI0NGYifQ==",
    "refreshToken": "eyJpZCI6IjQwNGEyNTdiYzM1YTQ1NDBhZWQwNzlkYzRiNDhkOTU3IiwidXNlcklkIjoiMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAiLCJ1c2VyTmFtZSI6bnVsbCwic2VjcmV0IjoiYjRhOWNhNDE2ODFlNGUyNjg4ZTU3NjI4ODdmZDE4MjEifQ==",
    "expire": 7200000,
    "failure": 86400000,
    "userInfo": {
      "id": "00000000000000000000000000000000",
      "tenantId": null,
      "deptId": null,
      "code": null,
      "name": "系统管理员",
      "account": "admin",
      "mobile": null,
      "email": null,
      "headImg": "http://image.insight.com/cloudfile/headimgs/default.png",
      "builtin": true,
      "createdTime": "2019-05-17 05:28:33"
    }
  },
  "option": null
}
```

[回目录](#目录)

### 验证Token

该接口可验证Token是否合法，无需传任何参数。

请求方法：**GET**

接口URL：**/base/auth/v1.0/tokens/status**

请求参数示例：

```java
@Test
public void testHttpCall() throws IOException {
    // given
    HttpGet request = new HttpGet("http://127.0.0.1:6200/base/auth/v1.0/tokens/verify");
    request.add("Accept", "application/json");
    request.add("Authorization","eyJpZCI6ImQxNzUwMjA5NzRlYjQ1MzJiY2U3MmY0NWRiZTkzMWYyIiwidXNlcklkIjoiMDAwMDAwMDAwMDAMDAwMDAwMDAwMDAwMDAwMDAwMDAiLCJ1c2VyTmFtZSI6bnVsbCwic2VjcmV0IjoiMTI5YTQ3MWRiMWYxNDA0DkxMzU5Y2JjNjcwYmE0NDQifQ==");

    // when
    HttpResponse response = HttpClientBuilder.create().build().execute(request);

    // then
    HttpEntity entity = response.getEntity();
    String jsonString = EntityUtils.toString(entity);
    loger.info(jsonString);
}
```

返回结果示例：

```json
{
  "success": true,
  "code": 200,
  "message": "请求成功",
  "data": null,
  "option": null
}
```

[回目录](#目录)

### 刷新Token

用户可在访问令牌过期后通过该接口刷新访问令牌的使用期限，访问令牌最多可刷新12次，延长过期时间24小时。用户调用接口成功后将获取一个新的访问令牌，原访问令牌失效，刷新令牌在失效前可一直用于刷新。此接口的限流策略为：同一设备的调用间隔需3秒以上，每天调用上限60次。

请求方法：**PUT**

接口URL：**/base/auth/v1.0/tokens**

接口返回数据类型：

|类型|属性|属性说明|
| ------------ | ------------ | ------------ |
|String|accessToken|访问用令牌|
|String|refreshToken|刷新用令牌|
|Integer|expire|令牌过期时间(毫秒)|
|Integer|failure|令牌失效时间(毫秒)|
|[UserInfo](#UserInfo)|userInfo|用户信息|

返回结果示例：

```json
{
  "success": true,
  "code": 200,
  "message": "请求成功",
  "data": {
    "accessToken": "eyJpZCI6IjQwNGEyNTdiYzM1YTQ1NDBhZWQwNzlkYzRiNDhkOTU3IiwidXNlcklkIjoiMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAiLCJ1c2VyTmFtZSI6bnVsbCwic2VjcmV0IjoiNzE5ODQ2MjU0NTY3NGNmY2I4MzRjZTkxYThjZTI0NGYifQ==",
    "refreshToken": "eyJpZCI6IjQwNGEyNTdiYzM1YTQ1NDBhZWQwNzlkYzRiNDhkOTU3IiwidXNlcklkIjoiMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAiLCJ1c2VyTmFtZSI6bnVsbCwic2VjcmV0IjoiYjRhOWNhNDE2ODFlNGUyNjg4ZTU3NjI4ODdmZDE4MjEifQ==",
    "expire": 7200000,
    "failure": 86400000,
    "userInfo": {
      "id": "00000000000000000000000000000000",
      "tenantId": null,
      "deptId": null,
      "code": null,
      "name": "系统管理员",
      "account": "admin",
      "mobile": null,
      "email": null,
      "headImg": "http://image.insight.com/cloudfile/headimgs/default.png",
      "builtin": true,
      "createdTime": "2019-05-17 05:28:33"
    }
  },
  "option": null
}
```

[回目录](#目录)

### 注销Token

用户调用该接口成功后，请求头所承载的访问令牌和与其对应的刷新令牌将被注销。

请求方法：**DELETE**

接口URL：**/base/auth/v1.0/tokens**

返回结果示例：

```json
{
  "success": true,
  "code": 200,
  "message": "请求成功",
  "data": null,
  "option": null
}
```

[回目录](#目录)

## 应用权限接口

### 获取模块导航

调用该接口可获取指定应用的模块导航数据。

请求方法：**GET**

接口URL：**/base/auth/v1.0/navigators**

接口返回数据类型：

|类型|属性|属性说明|
| ------------ | ------------ | ------------ |
|String|id|导航ID|
|String|parentId|父级导航ID|
|Integer|type|导航级别|
|Integer|index|索引,排序用|
|String|name|导航名称|
|[ModuleInfo](#ModuleInfo)|moduleInfo|模块信息|
|List\<[FuncDTO](#FuncDTO)>|functions|功能集合|

返回结果示例：

```json
{
  "success": true,
  "code": 200,
  "message": "请求成功",
  "data": [
    {
      "id": "8c95d6e097f340d6a8d93a3b5631ba39",
      "parentId": null,
      "type": 1,
      "index": 1,
      "name": "运营中心",
      "moduleInfo": {
        "icon": null,
        "iconUrl": null,
        "module": null,
        "file": null,
        "default": null
      },
      "functions": null
    },
    {
      "id": "711aad8daf654bcdb3a126d70191c15c",
      "parentId": "8c95d6e097f340d6a8d93a3b5631ba39",
      "type": 2,
      "index": 1,
      "name": "租户管理",
      "moduleInfo": {
        "icon": null,
        "iconUrl": null,
        "module": "Tenants",
        "file": "Base.dll",
        "default": true
      },
      "functions": null
    },
    {
      "id": "a65a562582bb489ea729bb0838bbeff8",
      "parentId": "8c95d6e097f340d6a8d93a3b5631ba39",
      "type": 2,
      "index": 2,
      "name": "应用管理",
      "moduleInfo": {
        "icon": null,
        "iconUrl": null,
        "module": "Apps",
        "file": "Base.dll",
        "default": false
      },
      "functions": null
    }
  ],
  "option": null
}
```

[回目录](#目录)

### 获取模块功能

接口功能描述(提供什么功能、影响什么数据、调用什么服务)

请求方法：**method**

接口URL：**/base/auth/v1.0/navigators/{id}/functions**

请求参数如下：

|类型|属性|是否必需|属性说明|
| ------------ | ------------ | ------------ | ------------ |
|String|id|是|模块ID(二级导航ID)|

接口返回数据类型：

|类型|属性|属性说明|
| ------------ | ------------ | ------------ |
|String|id|功能ID|
|String|navId|导航ID|
|Integer|type|节点类型|
|Integer|index|索引,排序用|
|String|name|功能名称|
|String|authCode|授权码|
|[IconInfo](#IconInfo)|iconInfo|功能图标信息|
|Boolean|permit|是否授权(true:已授权,false:已拒绝,null:未授权)|

返回结果示例：

```json
{
  "success": true,
  "code": 200,
  "message": "请求成功",
  "data": [
    {
      "id": "bfb02029cb0411e9bbd40242ac110008",
      "navId": "711aad8daf654bcdb3a126d70191c15c",
      "type": 0,
      "index": 1,
      "name": "刷新",
      "authCode": "getTenants",
      "iconInfo": {
        "icon": null,
        "iconUrl": null,
        "beginGroup": true,
        "hideText": true
      },
      "permit": null
    },
    {
      "id": "bfb02084cb0411e9bbd40242ac110008",
      "navId": "711aad8daf654bcdb3a126d70191c15c",
      "type": 0,
      "index": 2,
      "name": "新增租户",
      "authCode": "newTenant",
      "iconInfo": {
        "icon": null,
        "iconUrl": null,
        "beginGroup": true,
        "hideText": false
      },
      "permit": null
    },
    {
      "id": "bfb020cfcb0411e9bbd40242ac110008",
      "navId": "711aad8daf654bcdb3a126d70191c15c",
      "type": 1,
      "index": 3,
      "name": "编辑",
      "authCode": "editTenant",
      "iconInfo": {
        "icon": null,
        "iconUrl": null,
        "beginGroup": false,
        "hideText": false
      },
      "permit": null
    },
    {
      "id": "bfb021b2cb0411e9bbd40242ac110008",
      "navId": "711aad8daf654bcdb3a126d70191c15c",
      "type": 1,
      "index": 4,
      "name": "删除",
      "authCode": "deleteTenant",
      "iconInfo": {
        "icon": null,
        "iconUrl": null,
        "beginGroup": false,
        "hideText": false
      },
      "permit": null
    },
    {
      "id": "bfb021fecb0411e9bbd40242ac110008",
      "navId": "711aad8daf654bcdb3a126d70191c15c",
      "type": 1,
      "index": 5,
      "name": "绑定应用",
      "authCode": "bindApp",
      "iconInfo": {
        "icon": null,
        "iconUrl": null,
        "beginGroup": true,
        "hideText": false
      },
      "permit": null
    },
    {
      "id": "bfb02282cb0411e9bbd40242ac110008",
      "navId": "711aad8daf654bcdb3a126d70191c15c",
      "type": 1,
      "index": 6,
      "name": "续租",
      "authCode": "extend",
      "iconInfo": {
        "icon": null,
        "iconUrl": null,
        "beginGroup": false,
        "hideText": false
      },
      "permit": null
    }
  ],
  "option": null
}
```

[回目录](#目录)

## DTO类型说明

文档中所列举的类型皆为 **Java** 语言的数据类型，其它编程语言的的数据类型请自行对应。

### Reply

|类型|属性|属性说明|
| ------------ | ------------ | ------------ |
|Boolean|success|接口调用是否成功，成功：true；失败：false|
|Integer|code|错误代码，2xx代表成功，4xx或5xx代表失败|
|String|message|错误消息，描述了接口调用失败原因|
|Object|data|接口返回数据|
|Object|option|附加数据，如分页数据的总条数|

[回目录](#目录)

### UserInfo

|类型|属性|属性说明|
| ------------ | ------------ | ------------ |
|String|id|用户ID|
|String|tenantId|用户当前登录租户ID|
|String|deptId|用户当前登录部门ID|
|String|code|用户编码|
|String|name|用户姓名|
|String|account|用户登录账号|
|String|mobile|用户绑定手机号|
|String|email|用户绑定邮箱|
|String|headImg|用户头像|
|Boolean|builtin|是否内置用户|
|String|createdTime|用户创建时间|

[回目录](#目录)

### WeChatUser

|类型|属性|属性说明|
| ------------ | ------------ | ------------ |
|String|unionid|微信用户唯一ID|
|String|openid|微信公众号OpenId|
|String|nickname|微信昵称|
|String|sex|性别|
|String|country|所在国家|
|String|province|所在省/直辖市|
|String|city|所在地市|
|String|headimgurl|微信头像URL|
|List\<String>|privilege|微信用户特权|
|String|language|用户语言|

[回目录](#目录)

### ModuleInfo

|类型|属性|属性说明|
| ------------ | ------------ | ------------ |
|String|icon|图标|
|String|iconUrl|图标路径|
|String|module|模块名称|
|String|file|模块文件|
|Boolean|isDefault|是否启动模块|

[回目录](#目录)

### FuncDTO

|类型|属性|属性说明|
| ------------ | ------------ | ------------ |
|String|id|功能ID|
|String|navId|导航ID|
|Integer|type|节点类型|
|Integer|index|索引,排序用|
|String|name|功能名称|
|String|authCode|授权码|
|[IconInfo](#IconInfo)|iconInfo|功能图标信息|
|Boolean|permit|是否授权(true:已授权,false:已拒绝,null:未授权)|

[回目录](#目录)

### IconInfo

|类型|属性|属性说明|
| ------------ | ------------ | ------------ |
|String|icon|图标|
|String|iconUrl|图标路径|
|Boolean|isBeginGroup|是否开始分组|
|Boolean|isHideText|是否隐藏文字|

[回目录](#目录)
