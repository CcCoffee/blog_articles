# React中 useEffect 与 useCallback 的使用
useEffect 是 React hooks 的一个难点。本文通过真实案例引入 useEffect 和 useCallback 的概念和用法，试图让读者对 useEffect 的用法有个清晰的了解。

## 案例起因

登陆页面实现键盘监听，输入回车键触发登陆方法。

```javascript
function Login(props) {
  
  const [userName, setUserName] = useState('')
  const [password, setPassword] = useState('')
  const [isLoading, setIsLoading] = useState(false)

  function checkLogin(){
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
      withCredentials: true,
    }).then((resp) => {
      setIsLoading(false);
      if (resp.data.data === '登陆成功') {
        localStorage.setItem('token', resp.data.token)
        props.history.push('/console/')
      } else {
        message.error('用户名或密码错误')
      }
    })
  }
  
  // 新增按键事件监听
  document.onkeyup = function(event){
    var e=event||window.event;
    var keyCode = e.keyCode||e.which
    switch(keyCode){
      case 13: //enter键位编码
        checkLogin()
        break
      default:
        break;
    }
  }
}
```

成功触发登陆方法，进入首页。然而当在首页中继续按 `enter` 键时控制台出现下列错误：

```bash
index.js:1 Warning: Can't perform a React state update on an unmounted component. This is a no-op, but it indicates a memory leak in your application. To fix, cancel all subscriptions and asynchronous tasks in a useEffect cleanup function.
    in Login (created by Context.Consumer)
```

简单的来说就是无法在卸载一个组件之后继续更新 state 的状态值。这里是由于按键监听事件没有解除绑定，enter键最终触发了 checkLogin 方法调用了 setUserName 与 setPassword 更新 state。

解决方案：使用 useEffect 清理所有的订阅和异步任务。

##  什么是 `useEffect`

useEffect 中 `effect` 被翻译为**副作用**。 现在编写的是函数式组件，可以说是函数式编程 （虽然不完全是，但是是这样的味道）。函数式编程的特点就是无副作用，**输入输出一致性**。但是对于前端一些 Dom，Bom 等 API 来说，无副作用是不可能的，事件的绑定，定时器等等都，都是有副作用的。所以，为了处理这一部分的逻辑，React Hooks 提供了 useEffect 这个钩子来处理。所以说，我们看到的所有一些奇奇怪怪的地方，效果和理想不一致的情况，最终原因就是这个编程模式转变后，出现的"后遗症"。如果我们用函数式的思想来理解，这些问题都将会迎刃而解。

## 使用 useEffect 解除事件绑定

使用 useEffect 解除事件绑定就相当于解除了副作用。


```javascript
const onKeyDown = function(event){
  var e=event||window.event;
  var keyCode=e.keyCode||e.which
  switch(keyCode){
    case 13: //enter
      console.log("登陆")
      checkLogin()
      break
    default:
      break;
  }
}

useEffect(()=>{
  document.addEventListener('keydown', onKeyDown)
  return ()=>{
    document.removeEventListener('keydown', onKeyDown)
  }
},[])
```

重启应用，发现依然存在问题。下面的警告提示缺少依赖。


```bash
./src/pages/Login.jsx
  Line 74:5:  React Hook useEffect has a missing dependency: 'onKeyDown'. Either include it or remove the dependency array  react-hooks/exhaustive-deps
```

## useEffect 与 React 的渲染机制

为了更清楚的看清渲染的过程，添加日志输出

```react
function Login(props) {

  console.log("开始运行");

  const [userName, setUserName] = useState('')
  const [password, setPassword] = useState('')
  const [isLoading, setIsLoading] = useState(false)

  function checkLogin(){ ... }

  const onKeyDown = function(event){ ... }

  useEffect(()=>{
    console.log("绑定事件");
    document.addEventListener('keydown', onKeyDown)
    return ()=>{
      console.log("解绑事件");
      document.removeEventListener('keydown', onKeyDown)
    }
  }, [])

  console.log("一轮结束");

  return (
    <div className="login-div">
      <Spin tip="加载中..." spinning={isLoading}>
        <Card title="Kevin blog System" bordered={true} style={{ width: 400 }}>
          <form>
            <Input
              id="userName"
              size="large"
              placeholder="请输入用户名"
              name="username"
              autoComplete="username"
              prefix={<Icon type='user' style={{ color: 'rgba(0,0,0,.25)' }} />}
              onChange={(e) => setUserName(e.target.value)}
            />
            ...
          </form>
          ...
        </Card>
      </Spin>
    </div>
  )
```

刷新登陆页面，从下列日志可以看出 useEffect 是在页面渲染的最后才执行的。

```bash
开始运行
一轮结束
绑定事件
```

**每一个 state 变量的改变都会触发页面的重新渲染**

在用户名输入框中每输入一个字符，触发 setUserName 运行，进而出发页面重新渲染。值得注意的是这并不会触发上面 useEffect 方法，原因该 useEffect 方法并不依赖 userName 变量，代码上表现为在 useEffect 第二个参数中没有设置 userName 为它的依赖。

```
开始运行
一轮结束
```

为了证实这个说法，改写代码如下

```javascript
useEffect(()=>{
  console.log("绑定事件");
  document.addEventListener('keydown', onKeyDown)
  return ()=>{
    console.log("解绑事件");
    document.removeEventListener('keydown', onKeyDown)
  }
},[userName])
```

**每一个 useEffect 依赖项的值变化都会触发事件重新绑定**

在用户名输入框中每输入一个字符，都会输出下面的日志

```bash
开始运行
一轮结束
解绑事件
绑定事件
```

本质上来说**每一个 useEffect 中事件绑定方法所使用的 state 变量都有可能产生副作用**，绑定事件使用了闭包，不会因为 state 变量的改变而使用最新变量值，**如果绑定事件因为使用旧的变量值而产生错误，这被称为副作用**。避免副作用的方式就是在 state 变量改变时解除绑定，再绑定一个最新的闭包函数。

如果这么说你还不能理解，那么请看下面的例子：

```react
function Login(props) {
  const [testValue, setTestValue] = useState('old value')

  useEffect(()=>{
    const timeout1 = setTimeout(()=>{
      const date = new Date();
      console.log('时间 ' + date.getHours() + ':' + date.getSeconds() + ' > timeout1 设置值为 "new value"')
      setTestValue('new value')
    },1000)
    return () =>{
      console.log("清除 timeout1 定时任务")
      clearTimeout(timeout1)
    }
  },[])

  useEffect(()=>{
    const timeout2 = setTimeout(()=>{
      const date = new Date();
      console.log('时间 ' + date.getHours() + ':' + date.getSeconds() + ' > timetout2 获取的 testValue 值 : ' + testValue)
    },3000)
    return () =>{
      console.log("清除 timeout2 定时任务")
      clearTimeout(timeout2)
    }
  },[])
}
```

从下面的执行结果中可以看到，尽管timeout1已经把testValue值改变了，但是后执行的timeout2却没有办法拿到最新的值。本质上这是闭包引起的，设置 useEffect 的第二个参数是解决这种问题的手段，它能是的关注的某些 state 依赖变量改变时重新绑定事件。

```bash
开始运行
一轮结束
绑定事件

时间 11:30 > timeout1 设置值为 "new value"
开始运行
一轮结束
时间 11:32 > timetout2 获取的 testValue 值 : old value
```

为 useEffect 添加依赖变量

```javascript
useEffect(()=>{
  const timeout2 = setTimeout(()=>{
    const date = new Date();
    console.log('时间 ' + date.getHours() + ':' + date.getSeconds() + ' > timetout2 获取的 testValue 值 : ' + testValue)
  },3000)
  return () =>{
    console.log("清除 timeout2 定时任务")
    clearTimeout(timeout2)
  }
},[testValue])
```

从下面的输出日志可以知道，这回timeout2 可以获取到正确的新值了。

> 注意：这里由于定时任务重新绑定，导致了计时重置。

```bash
开始运行
一轮结束
绑定事件

时间 11:46 > timeout1 设置值为 "new value"
开始运行
一轮结束
清除 timeout2 定时任务
时间 11:49 > timetout2 获取的 testValue 值 : new value
```



回到我们之前的案例，react 贴心的提示如果没有为useEffect方法添加 `onKeyDown` 为依赖参数，会产生副作用。

```bash
./src/pages/Login.jsx
  Line 74:5:  React Hook useEffect has a missing dependency: 'onKeyDown'. Either include it or remove the dependency array  react-hooks/exhaustive-deps
```

这回我们可以很清楚的知道为什么要添加这个参数了

```javascript
useEffect(()=>{
  console.log("绑定事件");
  document.addEventListener('keydown', onKeyDown)
  return ()=>{
    console.log("解绑事件");
    document.removeEventListener('keydown', onKeyDown)
  }
},[onKeyDown])
```

看似完美地解决了。但是又发现了新的警告。每次创建的函数地址都是不同的。（言外之意就是，每一次的重新渲染，都会导致 onKeyDown 的重新绑定，每一个变量改变引起的重新渲染都会导致 onKeyDown 的更新）

```bash
webpackHotDevClient.js:137 ./src/pages/Login.jsx
  Line 70:9:  The 'onKeyDown' function makes the dependencies of useEffect Hook (at line 90) change on every render. Move it inside the useEffect callback. Alternatively, wrap the 'onKeyDown' definition into its own useCallback() Hook  react-hooks/exhaustive-deps
```

### `useCallback` 的使用

这时候可以通过 `useCallback` 用来缓存 onKeyDown 函数，其原理是通过缓存来达到不创建新的函数。onKeyDown 所依赖的 checkLogin 也要进行同样的改造。

```javascript
import React, { useState, useEffect, useCallback } from 'react'
function Login(props) {
  const checkLogin = useCallback(() => {
    ...
  },[userName, password, props.history])
  const onKeyDown = useCallback(function(event){
    var e=event||window.event;
    var keyCode=e.keyCode||e.which
    switch(keyCode){
      case 13: //enter
        checkLogin()
        break
      default:
        break;
    }
  },[checkLogin])
}
```

在某种程度上useEffect方式是用性能来换取函数式编程的规范。

## 参考链接

[函数式编程看React Hooks(二)事件绑定副作用深度剖析](https://blog.csdn.net/blueblueskyhua/article/details/102524335), by 蓝色的秋风
