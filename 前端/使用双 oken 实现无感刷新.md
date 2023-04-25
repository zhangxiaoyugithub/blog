> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/7224764099187736634)

前言
--

近期写的一个项目使用双token实现无感刷新。最后做了一些总结，本文详细介绍了实现流程，前后端详细代码。前端使用了Vue3+Vite，主要是axios封装，服务端使用了koa2做了一个简单的服务器模拟。

一、token 登录鉴权
------------

jwt：JSON Web Token。是一种认证协议，一般用来校验请求的身份信息和身份权限。 由三部分组成：Header、Hayload、Signature

header：也就是头部信息，是描述这个 token 的基本信息，json 格式

```
{
  "alg": "HS256", // 表示签名的算法，默认是 HMAC SHA256（写成 HS256）
  "type": "JWT" // 表示Token的类型，JWT 令牌统一写为JWT
}
复制代码
```

payload：载荷，也是一个 JSON 对象，用来存放实际需要传递的数据。不建议存放敏感信息，比如密码。

```
{
  "iss": "a.com", // 签发人
  "exp": "1d", // expiration time 过期时间
  "sub": "test", // 主题
  "aud": "", // 受众
  "nbf": "", // Not Before 生效时间
  "iat": "", // Issued At 签发时间
  "jti": "", // JWT ID 编号
  // 可以定义私有字段
  "name": "",
  "admin": ""
}
复制代码
```

Signature 签名 是对前两部分的签名，防止数据被篡改。 需要指定一个密钥。这个密钥只有服务器才知道，不能泄露。使用 Header 里面指定的签名算法，按照公式产生签名。

算出签名后，把 Header、Payload、Signature 三个部分拼成的一个字符串，每个部分之间用 . 分隔。这样就生成了一个 token

二、何为双 token
-----------

*   `accessToken`:用户获取数据权限
*   `refreshToken`:用来获取新的accessToken

双 token 验证机制，其中 accessToken 过期时间较短，refreshToken 过期时间较长。当 accessToken 过期后，使用 refreshToken 去请求新的 token。

#### 双 token 验证流程

1.  用户登录向服务端发送账号密码，登录失败返回客户端重新登录。登录成功服务端生成 accessToken 和 refreshToken，返回生成的 token 给客户端。
2.  在请求拦截器中，请求头中携带 accessToken 请求数据，服务端验证 accessToken 是否过期。token 有效继续请求数据，token 失效返回失效信息到客户端。
3.  客户端收到服务端发送的请求信息，在二次封装的 axios 的响应拦截器中判断是否有 accessToken 失效的信息，没有返回响应的数据。有失效的信息，就携带 refreshToken 请求新的 accessToken。
4.  服务端验证 refreshToken 是否有效。有效，重新生成 token， 返回新的 token 和提示信息到客户端，无效，返回无效信息给客户端。
5.  客户端响应拦截器判断响应信息是否有 refreshToken 有效无效。无效，退出当前登录。有效，重新存储新的 token，继续请求上一次请求的数据。

#### 注意事项

1.  短token失效，服务端拒绝请求，返回token失效信息，前端请求到新的短token如何再次请求数据，达到无感刷新的效果。
2.  服务端白名单，成功登录前是还没有请求到token的，那么如果服务端拦截请求，就无法登录。定制白名单，让登录无需进行token验证。

三、服务端代码
-------

#### 1. 搭建koa2服务器

全局安装koa脚手架

```
npm install koa-generator -g
复制代码
```

创建服务端 直接koa2+项目名

```
koa2 server
复制代码
```

cd server 进入到项目安装jwt

```
npm i jsonwebtoken
复制代码
```

为了方便直接在服务端使用koa-cors 跨域

```
npm i koa-cors
复制代码
```

在app.js中引入应用cors

```
const cors=require('koa-cors')
...
app.use(cors())
复制代码
```

#### 2. 双token

新建utils/token.js

```
const jwt=require('jsonwebtoken')

const secret='2023F_Ycb/wp_sd'  // 密钥
/*
expiresIn:5 过期时间，时间单位是秒
也可以这么写 expiresIn:1d 代表一天 
1h 代表一小时
*/
// 本次是为了测试，所以设置时间 短token5秒 长token15秒
const accessTokenTime=5  
const refreshTokenTime=15 

// 生成accessToken
const setAccessToken=(payload={})=>{  // payload 携带用户信息
    return jwt.sign(payload,secret,{expireIn:accessTokenTime})
}
//生成refreshToken
const setRefreshToken=(payload={})=>{
    return jwt.sign(payload,secret,{expireIn:refreshTokenTime})
}

module.exports={
    secret,
    setAccessToken,
    setRefreshToken
}
复制代码
```

#### 3. 路由

直接使用脚手架创建的项目已经在app.js使用了路由中间件 在router/index.js 创建接口

```
const router = require('koa-router')()
const jwt = require('jsonwebtoken')
const { getAccesstoken, getRefreshtoken, secret }=require('../utils/token')

/*登录接口*/
router.get('/login',()=>{
    let code,msg,data=null
    code=2000
    msg='登录成功，获取到token'
    data={
        accessToken:getAccessToken(),
        refreshToken:getReferToken()
    }
    ctx.body={
        code,
        msg,
        data
    }
})

/*用于测试的获取数据接口*/
router.get('/getTestData',(ctx)=>{
    let code,msg,data=null
    code=2000
    msg='获取数据成功'
    ctx.body={
        code,
        msg,
        data
    }
})

/*验证长token是否有效，刷新短token
  这里要注意，在刷新短token的时候回也返回新的长token，延续长token，
  这样活跃用户在持续操作过程中不会被迫退出登录。长时间无操作的非活
  跃用户长token过期重新登录
*/
router.get('/refresh',(ctx)=>{
    let code,msg,data=null
    //获取请求头中携带的长token
    let r_tk=ctx.request.headers['pass']
    //解析token 参数 token 密钥 回调函数返回信息
    jwt.verify(r_tk,secret,(error)=>{
        if(error){
            code=4006,
            msg='长token无效，请重新登录'
        } else{
            code=2000,
            msg='长token有效，返回新的token'，
            data={
                accessToken:getAccessToken(),
                refreshToken:getReferToken()
            }
        }
    })
})

复制代码
```

#### 4. 应用中间件

utils/auth.js

```
const { secret } = require('./token')
const jwt = require('jsonwebtoken')

/*白名单，登录、刷新短token不受限制，也就不用token验证*/
const whiteList=['/login','/refresh']
const isWhiteList=(url,whiteList)=>{
        return whiteList.find(item => item === url) ? true : false
}

/*中间件
 验证短token是否有效
*/
const cuth = async (ctx,next)=>{
    let code, msg, data = null
    let url = ctx.path
    if(isWhiteList(url,whiteList)){
        // 执行下一步
        return await next()
    } else {
        // 获取请求头携带的短token
        const a_tk=ctx.request.headers['authorization']
        if(!a_tk){
            code=4003
            msg='accessToken无效，无权限'
            ctx.body={
                code,
                msg,
                data
            }
        } else{
            // 解析token
            await jwt.verify(a_tk,secret.(error)=>{
                if(error)=>{
                      code=4003
                      msg='accessToken无效，无权限'
                      ctx.body={
                          code,
                          msg,
                          datta
                      }
                } else {
                    // token有效
                    return await next()
                }
            })
        }
    }
}
module.exports=auth
复制代码
```

在app.js中引入应用中间件

```
const auth=requier(./utils/auth)
···
app.use(auth)
复制代码
```

其实如果只是做一个简单的双token验证，很多中间件是没必要的，比如解析静态资源。不过为了节省时间，方便就直接使用了koa2脚手架。

最终目录结构：

![双token服务端目录结构.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/f64b40e3a7044a7fa3f462484ab927f9~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

四、前端代码
------

#### 1. Vue3+Vite框架

前端使用了Vue3+Vite的框架，看个人使用习惯。

```
npm init vite@latest client_side
复制代码
```

安装axios

```
npm i axios
复制代码
```

#### 2. 定义使用到的常量

config/ constants (恒) .js

```
export const ACCESS_TOKEN = 'a_tk' // 短token字段
export const REFRESH_TOKEN = 'r_tk' // 短token字段
export const AUTH = 'Authorization'  // header头部 携带短token
export const PASS = 'pass' // header头部 携带长token
复制代码
```

#### 3. 存储、调用过期请求

关键点：把携带过期token的请求，利用 promise (承诺) 存在数组中，保持pending状态，也就是不调用resolve()。当获取到新的token，再重新请求。 utils/ refresh (刷新) .js

```
export {REFRESH_TOKEN,PASS} from '../config/constants.js'
import { getRefreshToken, removeRefreshToken, setAccessToken, setRefreshToken} from '../config/storage'

let subsequent=[]
let flag=false // 设置开关，保证一次只能请求一次短token，防止客户多此操作，多次请求

/*把过期请求添加在数组中*/
export const addRequest = (request) => {
    subscribes.push(request)
}

/*调用过期请求*/
export const retryRequest = () => {
    console.log('重新请求上次中断的数据');
    subscribes.forEach(request => request())
    subscribes = []
}

/*短token过期，携带token去重新请求token*/
export const refreshToken=()=>{
    if(!flag){
        flag = true;
        let r_tk = getRefershToken() // 获取长token
        if(r_tk){
            server.get('/refresh',Object.assign({},{
                headers:{[PASS]=r_tk}
            })).then((res)=>{
                //长token失效，退出登录
                if(res.code===4006){
                    flag = false
                    removeRefershToken(REFRESH_TOKEN)
                } else if(res.code===2000){
                    // 存储新的token
                    setAccessToken(res.data.accessToken)
                    setRefreshToken(res.data.refreshToken)
                    flag = false
                    // 重新请求数据
                    retryRequest()
                }
            })
        }
    }
}
复制代码
```

#### 4. 封装axios

utlis/server.js

```
import axios from "axios";
import * as storage from "../config/storage"
import * as constants from '../config/constants'
import { addRequest, refreshToken } from "./refresh";

const server = axios.create({
    baseURL: 'http://localhost:3004', // 你的服务器
    timeout: 1000 * 10,
    headers: {
        "Content-type": "application/json"
    }
})

/*请求拦截器*/
server.interceptors.request.use(config => {
    // 获取短token，携带到请求头，服务端校验
    let aToken = storage.getAccessToken(constants.ACCESS_TOKEN)
    config.headers[constants.AUTH] = aToken
    return config
})

/*响应拦截器*/
server.interceptors.response.use(
    async response => {
        // 获取到配置和后端响应的数据
        let { config, data } = response
        console.log('响应提示信息：', data.msg);
        return new Promise((resolve, reject) => {
            // 短token失效
            if (data.code === 4003) {
                // 移除失效的短token
                storage.removeAccessToken(constants.ACCESS_TOKEN)
                // 把过期请求存储起来，用于请求到新的短token，再次请求，达到无感刷新
                addRequest(() => resolve(server(config)))
                // 携带长token去请求新的token
                refreshToken()
            } else {
                // 有效返回相应的数据
                resolve(data)
            }

        })

    },
    error => {
        return Promise.reject(error)
    }
)
复制代码
```

#### 5. 复用封装

```
import * as constants from "./constants"

// 存储短token
export const setAccessToken = (token) => localStorage.setItem(constanst.ACCESS_TOKEN, token)
// 存储长token
export const setRefershToken = (token) => localStorage.setItem(constants.REFRESH_TOKEN, token)
// 获取短token
export const getAccessToken = () => localStorage.getItem(constants.ACCESS_TOKEN)
// 获取长token
export const getRefershToken = () => localStorage.getItem(constants.REFRESH_TOKEN)
// 删除短token
export const removeAccessToken = () => localStorage.removeItem(constants.ACCESS_TOKEN)
// 删除长token
export const removeRefershToken = () => localStorage.removeItem(constants.REFRESH_TOKEN)
复制代码
```

#### 6. 接口封装

apis/index.js

```
import server from "../utils/server";
/*登录*/
export const login = () => {
    return server({
        url: '/login',
        method: 'get'
    })
}
/*请求数据*/
export const getData = () => {
    return server({
        url: '/getList',
        method: 'get'
    })
}
复制代码
```

项目运行
----

![双token前端.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/369fbd71f5c1454bac972990638cfee7~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

最后的最后，运行项目，查看效果 后端设置的短token5秒，长token10秒。登录请求到token后，请求数据可以正常请求，五秒后再次请求，短token失效，这时长token有效，请求到新的token， refresh (刷新) 接口只调用了一次。长token也过期后，就需要重新登录啦。 ![效果.png](https://p1-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/718dfbc173594c80969bbbcf543c643d~tplv-k3u1fbpfcp-zoom-in-crop-mark:1512:0:0:0.awebp?)

写在最后
----

这就是一整套的前后端使用双token机制实现无感刷新。token能做到的还有很多，比如权限管理、同一账号异地登录。本文只是浅显的应用了一下。

下次空余时间写写大文件切片上传，写文不易，大家多多点赞。感谢各位看官老爷。
