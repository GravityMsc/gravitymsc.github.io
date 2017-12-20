---
title: react-router和react-redux的权限认证
tags: [react,react-router,react-redux]
date: 2017-12-19 16:42:21
categories: 前端
---

>在前后端分离的情况下，路由由前端管理并控制，权限控制就成为了一个不可避免的问题

### 登录校验  
  就token验证来说，接口请求和路由控制都需要做出判定。由于react-router V4的组件化思想，取消了路由组件本身的钩子，例如: onEnter, onLeave 等，取而代之的是route组件的render函数或者是react组件的生命周期函数。下面分别用两种例子来实现路由的权限控制：  

  首先看官方示例，无状态组件作为函数，截取其核心部分：
```
  const AuthRoute = ({component: Component, ...rest}) => (
    <Route {...rest} render={(props) => (
      AuthFunction.isLogin ? <Component {...props} /> : 
        <Redirect to={{
          pathname: '/login',
          state: {from: props.location}
        }} />
    )} />
  );
```
  其次是react标准高阶组件形式，可扩展性更强，本质无区别：
```
  const withAuth = Component => (
    class extends React.Component {
      <!-- some operation -->
      render() {
        return (
          AuthFunction.isLogin ? <Component {...this.props} {...newProps} /> : 
            <Loading tips="请登录..." />
        )
      }
    }
  );
  <Route path="/private" exact component={withAuth(MainPage)} />
```

### 登录状态保存
  redux的状态管理，存储在内存中，在页面刷新后就会出现登录状态丢失，导致需要重新登录的情况。目前的处理措施分为两步：  
  - 在登录组件登录成功后把必要信息存储在localStroage或者sessionStorage中，做持久化保存。
  - 在根组件初始化加载过程中，通过redux dispatch,获取所存信息并保存在redux中，以便之后调用（在已有的fetch、axios之上做封装，每次执行拉取所存状态信息，与发送的实际参数合并，完成接口请求）

  对于单点登录，主要则参考掘金的这篇文章[《前端需要了解的 SSO 与 CAS 知识》](https://juejin.im/post/5a002b536fb9a045132a1727)

### browserRouter、hashRouter
  单页应用下，路由路径可以看作为状态信息。  
  若使用browserRouter，在后端未做特殊处理时，刷新页面因服务器寻找指定路径资源文件导致404错误。单页应用下，需要我们对后端加以限制。在请求其他路径资源失败时默认返回首页文件（多指index.html），以满足单页应用的路由选择；与此相对，hashRouter的优势在于不用对后端进行限制，路径以'#'分隔。在一些特殊情况下，我们可以利用history提供的监听函数，以便根据location决定当前情况下的一些状态变化。
  ```
  添加监听：
  this.listenHistory = history.listen((location) => {
    const selectedKey = location.pathname.split('/')[1];
    this.setState(() => ({
      current: selectedKey || 'jobMonitor',
    }));
  });
  解除监听：
  this.listenHistory();
  ```