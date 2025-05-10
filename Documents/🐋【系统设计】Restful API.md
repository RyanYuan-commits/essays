>[!summary] 总结：
REST 是资源表现形式转移的简称，根据 REST 提出了 RESTful 架构，客户端通过特定的 HTTP 动词实现对 URI 代表的资源进行操控；RESTful API 是根据 REST 构建的 API，提出了标准的规范，让用户能够通过路径 + Method 就能很清晰的理解路径的作用，通过 status_code 就能够理解接口的返回结果。
### 1 RESTful API 简介
#### 1.1 什么是 RESTful 架构？
REST（Resource Representational State Transfer），资源表现形式状态转移：
- 资源：真实的数据对象，每一种资源都有特定的统一资源标识符 URI 与之对应；
- 表现形式：资源的外在表现格式，比如 `json`、`image`、`txt` 等；
- 状态转移：请求服务器引发的资源状态的改变，常见的就是增删改查。
总结一下什么是 RESTful 架构就是，每一个 URI 代表一种资源；客户端和服务器之间，传递这种资源的某种表现形式比如 `json`，`xml`，`image`,`txt` 等等；客户端通过特定的 HTTP 动词，对服务器端资源进行操作，实现"表现层状态转化"。
#### 1.2 REST API 是什么？
**RESTful API** 经常也被叫做 **REST API**，它是基于 REST 构建的 API。
```api
GET    /classes：列出所有班级
POST   /classes：新建一个班级
```
RESTful API 的优点是通过 URL + Method 就能确定这个 URL 的作用，通过 status_code 就能知道请求结果如何。
### 2 RESTful API 规范
#### 2.1 动作规范
- `GET`：请求从服务器获取特定资源。举个例子：`GET /classes`（获取所有班级）
- `POST`：在服务器上创建一个新的资源。举个例子：`POST /classes`（创建班级）
- `PUT`：更新服务器上的资源，客户端提供更新后的整个资源。举个例子：`PUT /classes/12`（更新编号为 12 的班级）
- `DELETE`：从服务器删除特定的资源。举个例子：`DELETE /classes/12`（删除编号为 12 的班级）
- `PATCH`：更新服务器上的资源，客户端提供更改的属性，可以看做作是部分更新，使用的比较少。
#### 2.2 路径规范
路径又称"终点"（endpoint），表示 API 的具体网址。实际开发中常见的规范如下：
1. **网址中不能有动词，只能有名词，API 中的名词也应该使用复数**：因为 REST 中的资源往往和数据库中的表对应，而数据库中的表都是同种记录的"集合"。如果 API 调用并不涉及资源（如计算，翻译等操作）的话，可以用动词。比如：`GET /calculate?param1=11&param2=33` 。
2. **不用大写字母，建议用 `-` 不用 `_`**：比如邀请码写成 `invitation-code`而不是 ~~invitation_code~~ 。
3. **善用版本化 API**：当我们的 API 发生了重大改变而不兼容前期版本的时候，我们可以通过 URL 来实现版本化，比如 `http://api.example.com/v1`、`http://apiv1.example.com` 。版本不必非要是数字，只是数字用的最多，日期、季节都可以作为版本标识符，项目团队达成共识就可。
4. **接口尽量使用名词，避免使用动词**：RESTful API 操作（HTTP Method）的是资源（名词）而不是动作（动词）。
#### 2.3 过滤信息规范
果我们在查询的时候需要添加特定条件的话，建议使用 url 参数的形式：
```c
GET    /classes?state=active&name=guidegege
GET    /classes?page=1&size=10
```
#### 2.4 状态码规范
![[HTTP 状态码规范.png|600]]