---
layout: post
title: RESTful API设计指南
description: RESTful介绍及API设计规范
category: blog
---

## 前言

API（Application Programming Interface），说白了就是一个软件对外提供的能力的表征，是外界使用这些能力的钥匙。对于一个系统能力各方面都OK的服务来说，API设计不好的结果，就像一个满脑子干货却因表达不佳只能接受外界的冷眼甚至误解的倒霉蛋一样，悲催。不过，由于它看起来并不是那么high-tech，所以很多开发人员其实并没有将其真正放在心上，也缺少从使用者的视角审视自身API合理性的认知习惯。软件工程毕竟是门协作的艺术，我们有义务让自己交付的服务接口（代码如是）能够直观准确地体现其赋予的能力。

个人认为，一个设计良好的API，起码具备以下三点：

- 清楚准确地告知他人其所具备的能力（有啥用）
- 明确定义使用该能力的前提条件（怎么用，涉及方法、参数、鉴权等）
- 提供各种情形下的响应信息和处理方式（结果如何，涉及状态码、错误信息等）

具体到怎么设计，从RESTful的角度来理解，或许可以找到一个相对满意的答案。



## 什么是RESTful

RESTful（Representational State Transfer）是一种基于HTTP的、**围绕资源（Resource）进行组织的设计风格**，它提供了一组设计原则和约束条件，并不是一种标准。它可以清晰地告诉别人这个接口是做什么的（具有自解释性），与传统的API设计相比，它更简洁、更有层次，也更容易理解和使用。

具体该如何来设计一个优雅、规范的RESTful API呢？我们可以带着以下几个问题，继续深入：

- 项目资源的URL应该如何设计？用名词复数还是单数？一个资源需要多少个URL？
- 用哪种HTTP方法来操作一个资源？
- 那些不涉及资源操作的URL怎么办？
- 可选、复杂的参数应该放在哪里？
- 实现分页和版本控制的方法是什么？
- 如何正确地使用HTTP状态码？
- 什么样的错误信息是准确有用的？



## RESTful API设计规范

### 使用HTTPS

这个和 RESTful API 本身没有很大的关系，但是对于增加网站的安全是非常重要的。特别是，如果你提供的是公开 API，用户的信息泄露或者被攻击会严重影响网站的信誉。

PS：不要让非SSL的url访问重定向到SSL的url。

### API地址和版本

在 URL 中指定 API 的版本是个很好的做法。如果你有不兼容和破坏性的更改，版本号将让你能更容易地发布API。发布新API时，只需要增加版本号中的数字。这样的话，客户端可以自如地迁移到新API，不会因调用完全不同的新API而陷入困境。 

使用直观的 “v” 前缀来表示后面的数字是版本号，例如：

```shell
/v1/employees
```

另外，你不需要使用次级版本号（“v1.2”），因为你不应该频繁地去发布API版本。

### 以资源为中心的URL设计

资源是 RESTful API 的核心元素，所有的操作都是针对特定资源进行的。URL用于指定你要用的资源。

Github 可以说是这方面的典范，下面以 repository 来说明：

```shell
/users/:username/repos
/users/:org/repos
/repos/:owner/:repo
/repos/:owner/:repo/tags
/repos/:owner/:repo/branches/:branch
```

我们可以看到几个特性：

- 资源分为单个文档和集合，尽量使用复数来表示资源，单个资源通过添加 id 或者 name 等来表示。
- 一个资源可以有多个不同的 URL。
- 资源可以嵌套，通过类似目录路径的方式来表示，以体现它们之间的关系。

用名词代替动词表示资源，这会让API更简洁，URL数目更少，例子如下：

```shell
# bad design
/getAllEmployees
/getAllExternalEmployees
/createEmployee
/updateEmployee

# better design
GET /employees
GET /employees?state=external
POST /employees
PUT /employees/56
```

另外：

- 根据RFC3986定义，URL是大小写敏感的。所以为了避免歧义，尽量使用小写字母。
- 避免复数和单数名词混合使用，这显得非常混乱且容易出错。
- 避免层级过深的URI，过深的导航容易导致URL膨胀，不易维护。例如 `GET /zoos/1/areas/3/animals/4`，尽量使用查询参数代替路径中的实体导航，如 `GET /animals?zoo=1&area=3`。

### 使用正确的HTTP Method

有了资源的 URL 设计，所有针对资源的操作都是使用 HTTP 方法指定的。比较常用的方法有：

| Method | Description                                                  |
| ------ | ------------------------------------------------------------ |
| GET    | 获取资源。从不改变资源的状态，无副作用。                     |
| POST   | 创建新资源。                                                 |
| PUT    | 更新现有资源。                                               |
| PATCH  | 更新资源的部分属性。因为 PATCH 比较新，而且规范比较复杂，所以真正实现的比较少，一般都是用 POST 替代。 |
| DELETE | 删除现有资源。                                               |
| HEAD   | 只获取某个资源的头部信息。比如只想了解某个文件的大小、某个资源的修改日期等。 |

示例如下：

```
GET /repos/:owner/:repo/issues
GET /repos/:owner/:repo/issues/:number
POST /repos/:owner/:repo/issues
PATCH /repos/:owner/:repo/issues/:number
DELETE /repos/:owner/:repo
```

另外，更新和创建操作应该返回最新的资源，来通知用户资源的情况；删除资源一般不会返回资源内容。

### 不符合CRUD的情况

在实际资源操作中，总会有一些不符合 CRUD（Create-Read-Update-Delete） 的情况，或者调用并不涉及资源（如计算、翻译、转换等），一般有几种处理方法。

#### 1. 使用POST

为需要的动作增加一个 endpoint，使用 POST 来执行动作并将结果返回给客户端，比如 `POST /resend` 重新发送邮件。这种时候建议在URL中使用动词而不是名词，以便清楚地区分资源请求和非资源请求。

#### 2. 增加控制参数

添加动作相关的参数，通过修改参数来控制动作。

比如一个博客网站，会有把写好的文章“发布”的功能，可以用 `POST /articles/{:id}/publish` 方法，也可以在文章中增加 published:boolean 字段，发布的时候就是更新该字段 `PUT /articles/{:id}?published=true`。

#### 3. 把动作转换成资源

把动作转换成可以执行 CRUD 操作的资源。

以Github为例：

- “喜欢”一个 gist，就增加一个 `/gists/:id/star` 子资源，然后对其进行操作：“喜欢”使用 PUT `/gists/:id/star`，“取消喜欢”使用 `DELETE /gists/:id/star`。 

- Fork也是一个动作，但是在 gist 下面增加 forks资源，就能把动作变成 CRUD 兼容的：`POST /gists/:id/forks` 可以执行用户 fork 的动作。 

### Query让查询更自由

为了让你的URL更小、更简洁，对可选的、复杂的参数，使用查询字符串（?）。

以Github为例，比如查询某个 repo 下面 issues 的时候，可以通过以下参数来控制返回哪些结果：

- state：issue 的状态，可以是 open，closed，all 
- since：在指定时间点之后更新过的才会返回
- assignee：被 assign 给某个 user 的 issues
- sort：选择排序的值，可以是 created、updated、comments 
- direction：排序的方向，升序（asc）还是降序（desc）

其他示例如下：

```shell
# bad design
GET /employees
GET /externalEmployees
GET /internalEmployees
GET /internalAndSeniorEmployees

# better design
GET /employees?state=internal&maturity=senior
```

另外，经常使用的、复杂的查询标签化，降低维护成本。例如：

```shell
# before
GET /trades?status=closed&sort=created,desc

# after
GET /trades/recently-closed
```

### 分页机制

当返回某个资源的列表时，如果要返回的数目特别多，比如 Github 的 `/users`，就需要使用分页机制分批次按照需要来返回特定数量的结果。一次性返回数据库中所有资源不是一个好主意。

分页的实现会用到上面提到的 url query，通过两个参数来控制要返回的资源结果：

- page：要获取哪一页的资源，默认是第一页。

- pageSize：每页返回多少资源，如果没提供会使用预设的默认值；这个数量也是有一个最大值，不然用户把它设置成一个非常大的值（比如 99999999）也失去了设计的初衷。

```shell
/employees?page=1&pageSize=10
```

返回的资源列表为 [(page-1) * pageSize, page * pageSize)。Github API 文档中还提到一个很好的点，相关的分页信息还可以存放到 Link 头部，这样客户端可以直接得到诸如下一页、最后一页、上一页等内容的URL地址，而不是自己手动去计算和拼接。 

另外，如果你使用了分页，客户端需要知道资源总数，例如以下响应内容：

```json
{
  "page": 1,
  "pageSize": 10,
  "total": 2048,
  "employees": [
    // ...
  ]
}
```

### 响应返回的Schema

JSON 因为它的可读性、紧凑性以及多种语言支持等优点，成为了 HTTP API 最常用的返回格式。如果用户需要其他格式，比如 xml，应该在请求头部 Accept 中指定。对于不支持的格式，服务端需要返回正确的 status code，并给出详细的说明。

### 选择合适的HTTP状态码

HTTP 应答中，需要带一个很重要的字段：status code。它说明了请求的大致情况，是否正常完成、需要进一步处理、出现了什么错误等。状态码都是三位的整数，大概分成了几个区间：

- 2XX：请求正常处理并返回。
- 3XX：重定向，请求的资源位置发生变化。
- 4XX：客户端发送的请求有错误。
- 5XX：服务器端错误。

在 HTTP API 设计中，经常用到的状态码以及它们的意义如下表：

| Code | Label                  | Description                                                  |
| ---- | ---------------------- | ------------------------------------------------------------ |
| 200  | OK                     | 请求成功接收并处理，一般响应中都会有 body。                  |
| 201  | Created                | 请求已完成，并导致了一个或者多个资源被创建，最常用在 POST 创建资源的时候。 |
| 202  | Accepted               | 请求已经接收并开始处理，但是处理还没有完成。一般用在异步处理的情况，响应 body 中应该告诉客户端去哪里查看任务的状态。 |
| 204  | No Content             | 请求已经处理完成，但是没有信息要返回，经常用在 PUT 更新资源的时候（客户端提供资源的所有属性，因此不需要服务端返回）。如果有重要的 metadata，可以放到头部返回。 |
| 301  | Moved Permanently      | 请求的资源已经永久性地移动到另外一个地方，后续所有的请求都应该直接访问新地址。服务端会把新地址写在 Location 头部字段，方便客户端使用。允许客户端把 POST 请求修改为 GET。 |
| 304  | Not Modified           | 请求的资源和之前的版本一样，没有发生改变。用来缓存资源，和条件性请求（conditional request）一起出现。 |
| 307  | Temporary Redirect     | 目标资源暂时性地移动到新的地址，客户端需要去新地址进行操作，但是**不能**修改请求的方法。 |
| 308  | Permanent Redirect     | 和 301 类似，除了客户端**不能**修改原请求的方法。            |
| 400  | Bad Request            | 客户端发送的请求有错误（请求语法错误、body 数据格式有误、body 缺少必须的字段等），导致服务端无法处理。 |
| 401  | Unauthorized           | 请求的资源需要认证，客户端没有提供认证信息或者认证信息不正确。 |
| 403  | Forbidden              | 服务器端接收到并理解客户端的请求，但是客户端的权限不足。比如，普通用户想操作只有管理员才有权限的资源。 |
| 404  | Not Found              | 客户端要访问的资源不存在，链接失效或者客户端伪造 URL 的时候回遇到这个情况。 |
| 405  | Method Not Allowed     | 服务端接收到了请求，而且要访问的资源也存在，但是不支持对应的方法。服务端**必须**返回 Allow 头部，告诉客户端哪些方法是允许的 。 |
| 415  | Unsupported Media Type | 服务端不支持客户端请求的资源格式，一般是因为客户端在 Content-Type 或者 Content-Encoding 中申明了希望的返回格式，但是服务端没有实现。比如，客户端希望收到xml返回，但是服务端支持Json。 |
| 429  | Too Many Requests      | 客户端在规定的时间里发送了太多请求，在进行限流的时候会用到。 |
| 500  | Internal Server Error  | 服务器内部错误，导致无法完成请求的内容。                     |
| 503  | Service Unavailable    | 服务器因为负载过高或者维护，暂时无法提供服务。服务器端应该返回 Retry-After 头部，告诉客户端过一段时间再来重试。 |

更完整的列表可以参考 [HTTP Status Code](https://httpstatuses.com/) 这个网站。

另外，**建议尽量使用HTTP状态码来表征返回结果，而非发生错误的时候返回一个HTTP 200响应，然后在response里包装一个自定义的code。这样做的好处除了降低调用方的认知负担之外，还有利于做请求的监控。**

### 返回准确有用的错误信息

如果出错的话，在 response body 中通过 `message` 给出明确的信息。 

比如客户端发送的请求有错误，一般会返回 4XX Bad Request 结果。这个结果很模糊，给出错误 message 的话，能更好地让客户端知道具体哪里有问题，并进行快速修改。 

- 如果请求的 JSON 数据无法解析，会返回 Problems parsing JSON。 
- 如果缺少必要的 field，会返回 422 Unprocessable Entity，除了 message 之外，还可以通过 errors 给出了哪些 field 缺少了，能够方便调用方快速排错。 

基本的思路就是尽可能提供更准确的错误信息：比如数据不是正确的 json、缺少必要的字段、字段的值不符合规定…… 而不是直接说“请求错误”之类的信息。

响应示例：

```json
{
  "code": 400,
  "message": "Method argument not valid. Argument 'internal' is neccesary.",
  "data": {}
}
```

另外，**针对微服务，个人认为：**

- **如果是直接对外的API，可以设计统一的response格式，接受适当包装，但code不要自定义。对业务类异常，用其对应的HTTP状态码；对非业务类异常，统一500。**
- **如果是只提供给系统其他服务调用的API，response直接返回数据，不需要做多余的包装。**

### 认证与授权

一般来说，让任何人随意访问公开的 API 是不好的做法。认证和授权是两件事情：

- 认证（Authentication）是为了确定用户是其申明的身份，比如提供账户的密码。
- 授权（Authorization）是为了保证用户有对请求资源特定操作的权限。比如用户的私人信息只能自己能访问，其他人无法看到；有些特殊的操作只能管理员可以操作，其他用户有只读的权限等等。

如果没有通过认证（提供的用户名和密码不匹配，token 不正确等），需要返回 401 Unauthorized 状态码，并在 body 中说明具体的错误信息；而没有被授权访问的资源操作，需要返回 403 Forbidden 状态码，还有详细的错误信息。 

PS：Github API 对某些用户未被授权访问的资源操作返回 404 Not Found ，目的是为了防止私有资源的泄露（比如黑客可以自动化试探用户的私有资源，返回 403 的话，就等于告诉黑客用户有这些私有的资源）。 

### 限流

如果对访问的次数不加控制，很可能会造成 API 被滥用，甚至被 DDos 攻击。根据使用者不同的身份对其进行限流，可以防止这些情况，减少服务器的压力。 

对用户的请求限流之后，要有方法告诉用户它的请求使用情况，Github API 使用的三个相关的头部： 

- `X-RateLimit-Limit`：用户每个小时允许发送请求的最大值。
- `X-RateLimit-Remaining`：当前时间窗口剩下的可用请求数目。
- `X-RateLimit-Rest`：时间窗口重置的时候，到这个时间点可用的请求数量就会变成 `X-RateLimit-Limit` 的值。

如果允许没有登录的用户使用 API（可以让用户试用），可以把 `X-RateLimit-Limit` 的值设置得很小，比如 Github 使用的 60。没有登录的用户是按照请求的 IP 来确定的，而登录的用户按照认证后的信息来确定身份。 

对于超过流量的请求，可以返回 429 Too many requests 状态码，并附带错误信息。而 Github API 返回的是 403 Forbidden ，虽然没有 429 更准确，也是可以理解的。

### Hypermedia API

RESTful API 的设计最好做到 Hypermedia，即在返回结果中提供相关资源的链接。这种设计也被称为 HATEOAS。这样做的好处是，用户可以根据返回结果就能得到后续操作需要访问的地址。 

比如访问 [api.github.com](https://api.github.com/)，就可以看到 Github API 支持的资源操作。 

另外，这种做法一是可以让客户端更好地应对一些API变更的情况，二是能让你的API变得可以自我描述，需要写的文档更少。例如：

```json
{
  "id": 1,
  "name": "Brian",
  "links": [
    {
      "ref": "salary",
      "href": "/employees/1/salary" // 客户端将始终获得一个有效的URL，即使你对其进行了变更
    }
  ]
}
```

### 编写优质的文档

API 最终是给人使用的，不管是公司内部，还是公开的 API 都是一样。即使我们遵循了上面提到的所有规范，设计的 API 非常优雅，用户还是不知道怎么使用我们的 API。所以，最后一步，但非常重要的一步是：为你的 API 编写优质的文档。

对每个请求以及返回的参数给出说明，最好给出一个详细而完整的示例，提醒用户需要注意的地方。总之，**目标就是用户可以根据你的文档就能直接使用 API，而不是要发邮件给你，或者跑到你的座位上问你一堆问题。**



## 参考资料

- [跟着 Github 学习 Restful HTTP API 设计](https://cizixs.com/2016/12/12/restful-api-design-guide/)
- [[译] RESTful API 设计最佳实践](https://juejin.im/entry/59e460c951882542f578f2f0)
- [Restful API 的设计规范](http://novoland.github.io/%E8%AE%BE%E8%AE%A1/2015/08/17/Restful%20API%20%E7%9A%84%E8%AE%BE%E8%AE%A1%E8%A7%84%E8%8C%83.html)
- [REST的提出：Roy Thomas Fielding 的博士论文](https://www.ics.uci.edu/~fielding/pubs/dissertation/top.htm)

