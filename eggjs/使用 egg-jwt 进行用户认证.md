# 使用 egg-jwt 进行用户认证
本文将介绍如何使用 egg.js 提供的 egg-jwt 插件保护应用接口。
> JWT 基于对称加密的原理使得应用无需再保存用户登陆状态。通过私钥解密用户传过来的 Token，若成功解密且发现未过过期时间则可以认为处于合法登陆状态。

# 案例介绍
本案例中 admin 项目为后台管理系统，service 项目则为 admin 提供数据操作的接口。service 对除登陆鉴定的接口外的所有接口进行 用户登陆凭据（JWT）鉴定，只有在 JWT 合法的情况下用户才能调用 service 提供的业务数据接口。

**项目目录结构** ：
```bash
.
├── admin
│   ├── README.md
│   ├── build
│   ├── node_modules
│   ├── package.json
│   ├── public
│   ├── src
│   ├── start.sh
│   └── yarn.lock
└── service
    ├── README.md
    ├── app
    ├── appveyor.yml
    ├── config
    ├── jsconfig.json
    ├── logs
    ├── node_modules
    ├── package.json
    ├── resource
    ├── run
    ├── test
    ├── typings
    └── yarn.lock
```

## 使用 egg-jwt  保护 service 的接口
### 引入依赖

`yarn add egg-jwt`

### 修改配置
```bash
config
├── config.default.js
├── config.dev.js
├── config.prod.js
└── plugin.js
```
config.dev.js
```javascript
'use strict';

module.exports = appInfo => {
  const userConfig = exports = {};
  // 略
  userConfig.jwt = {
    secret: '123456',
  };
  return {
    ...userConfig,
  };
};
```

plugin.js
```javascript
'use strict';
module.exports = {
  jwt: {
    enable: true,
    package: 'egg-jwt',
  },
}
```
### 代码应用
**router/admin.js**
```javascript
'use strict';
module.exports = app => {
  const {
    jwt,
    router,
    controller,
  } = app;

  router.post('/admin/checkLogin', controller.admin.main.checkLogin); //       用户登陆接口放行
  router.get('/admin/articles', jwt, controller.admin.main.articles); //       对需要保护的业务接口使用 egg-jwt 插件拦截处理
};
```
**app/controller/admin/src/main.js**
```javascript
'use strict';
const Controller = require('egg').Controller;
class MainController extends Controller {
  async checkLogin() {
    const {
      ctx,
      app,
    } = this;
    const user_name = ctx.request.body.user_name;
    const password = ctx.request.body.password;
    const selectUserSQL = 'SELECT * FROM user WHERE user_name = \'' + user_name + '\' AND password = \'' + password + '\'';
    const users = await app.mysql.query(selectUserSQL);
    if (users.length > 0) {
      // 通过JWT的方式限制session过期时间为30分钟
      ctx.body = {
        code: 1,
        message: '登陆成功',
        token: app.jwt.sign({
          exp: Math.floor(Date.now() / 1000) + (app.config.tokenExpireTime * 60),
          user: user_name,
        },
        app.config.jwt.secret),
      };
    } else {
      ctx.body = {
        message: '登陆失败',
      };
      ctx.status = 401;
    }
  }
}
```

### 使用 postman 测试接口
**访问受保护的接口**

使用 postman 通过 GET 请求访问`http://localhost:7001/admin/articles`

请求头
```json
{
     "Accept": "application/json"
}
```

响应
```json
{
   "code": "credentials_required",
  "message": "No authorization token was found"
}
```
**用户登陆获取token**

使用 postman 通过 POST 请求访问`http://localhost:7001/admin/checkLogin`

请求头
```json
{
  "Content-Type": "application/json; charset=utf-8"
}
```

请求参数 : `{"user_name":"kevin","password":"123456"}`

响应
```json
{
    "message": "登陆成功",
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE1ODQ0NTM5ODYsInVzZXJfbmFtZSI6ImtldmluIiwiaWF0IjoxNTg0NDUzOTI2fQ.FtqUskpmu7CZccGjzZuE8TIMZYNLNYxERulko4_pPN8"
}
```
  
**使用token访问受保护接口**
使用 postman 通过 GET 请求访问`http://localhost:7001/admin/articles`

请求头
```json
{
     "Accept:" "application/json"
     "Authorization": "Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE1ODQ0NTM5ODYsInVzZXJfbmFtZSI6ImtldmluIiwiaWF0IjoxNTg0NDUzOTI2fQ.FtqUskpmu7CZccGjzZuE8TIMZYNLNYxERulko4_pPN8"
}
```
响应
```json
{
    "data": [
      {
        "id": 16,
        "type_name": "java",
        "title": "待写文章清单",
      }
    ]
 }
```
## admin 应用访问 service 受保护接口
与业务无关的代码我们希望能够抽离出去，这里通过axios的请求拦截器把 token 放置于请求头中; 通过响应拦截器让token过期后能够跳转到登陆页面。

### 修改用户登陆逻辑

**src/pages/Login.jsx**
```javascript
function Login(props) {

  const [userName, setUserName] = useState('')
  const [password, setPassword] = useState('')
  const [isLoading, setIsLoading] = useState(false)

  const checkLogin = () => {
    setIsLoading(true)
    if (!userName) {
      setTimeout(() => setIsLoading(false), 500)
      message.error('用户名不能为空')
      return false
    } else if (!password) {
      setTimeout(() => setIsLoading(false), 500)
      message.error('密码不能为空')
      return false
    }
    axios({
      method: 'POST',
      url: API.checkLogin,
      data: {
        user_name: userName,
        password
      },
      // 普通的ajax请求浏览器是默认携带cookie，但是跨域请求默认是不携带cookie的。开启withCredentials后服务端才能拿到cookie
      withCredentials: true,
    }).then((resp) => {
      setIsLoading(false);
      if (resp.data.code === 1) {
        localStorage.setItem('token', resp.data.token)
        // 编程导航
        props.history.push('/console/')
      } else {
        message.error('用户名或密码错误')
      }
    })
}
```
### 添加 HTTP 客户端 axios 请求与响应拦截器

**src/utils/axios.js**
```javascript
/**
 * 通过包装axios进行个性化定制，这里实现了自动添加token的功能
 */
import axios from 'axios'
import { message } from 'antd'

axios.interceptors.request.use(config=> {
  let token = localStorage.getItem('token')
  if (token) {
    config.headers.Authorization = "Bearer "+token
  }
  return config
}, error=> {
  console.info("error: ");
  console.info(error);
  return Promise.reject(error);
})

// 添加一个响应拦截器
axios.interceptors.response.use(function (response) {
  return response;
}, async function (error) { // async 可以添加在任何function前
  if (error.response) {
    switch (error.response.status) {
      case 401:
        // 返回 401 清除token信息并跳转到登录页面
        localStorage.removeItem('token')
        message.error("会话已过期，请重新登陆")
        await sleep(2000)
        window.location.replace('/');
    }
  }
  return Promise.reject(error.response.data)   // 返回接口返回的错误信息
})
const sleep = (duration) => new Promise((res, rej) => setTimeout(res, duration));

export default axios
```
