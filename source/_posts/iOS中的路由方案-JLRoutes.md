---
title: iOS中的路由方案 -- JLRoutes
date: 2017-05-13 21:05:46
tags: iOS
---

## 路由由来

路由层的概念在服务端是指url请求的分层解析，将一个请求分发到对应的应用处理程序。前端路由的主要作用是保证视图和URL的同步。移动端的路由层指的是将诸如App内页面访问、H5与App访问的访问请求和App间的访问请求，进行分发处理的逻辑层。

## 路由主要应用场景

1、App之间的相互跳转访问
2、App内部页面跳转、模块调度与组件加载等
3、H5页面与App原生页面、模块与组件的交互
4、更好的兼容iOS、Android的系统访问机制、App链接协议、web端路由机制与前端开发规范等，兼容各平台(Android、iOS)App页面导航机制
5、以组件化为目的的工程改造，隔离各个业务，以制作单独的组件
6、Extension等动态调用主App的资源

## JLRoutes

JLRoutes是一个调用极少代码 , 可以很方便的处理不同URL schemes以及解析它们的参数，并通过回调block来处理URL对应的操作 , 可以用于处理复杂跳转逻辑的三方库。
JLRoutes可以看做是一个路由调度器，通过注册路由地址（URL），进而实现对不同的URL做出反应，最重要的是它可以帮助我们解析出URL中的所有参数，只需要在注册的时候定义一些key，就可以在Block中通过这些key拿到对应的参数value。
工作原理：利用UrlScheme做跳转，使用Url传递参数，打开指定的App或者某一个页面。

## 使用步骤

可以通过CocoaPods把JLRoutes集成到项目中。

### 一、配置URL Schemes

在工程info.plist文件中进行添加：
配置info.plist —>添加URL types字段 , 你会发现其为一个数组 –>然后咱们在item 0 处在加一个URL Schemes , 又是一个数组，一个app可以对应多个scheme —>配置scheme , URL identifier 最好设置复杂些 , 保证其唯一性。

### 二、注册

注册方式如下：

```
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
  //注册全局JLRoutes
  JLRoutes *routes = [JLRoutes globalRoutes];
  [routes addRoute:@"/user/view/:userID" handler:^BOOL(NSDictionary *parameters) {
    NSString *userID = parameters[@"userID"]; // defined in the route by specifying ":userID"

    // present UI for viewing user with ID 'userID'

    return YES; // return YES to say we have handled the route
  }];
  //
  return YES;
}
//实现跳转拦截
- (BOOL)application:(UIApplication *)application openURL:(NSURL *)url sourceApplication:(NSString *)sourceApplication annotation:(id)annotation
{
  return [JLRoutes routeURL:url];
}
```

也可以通过如下方式注册:

```
JLRoutes.globalRoutes[@"/route/:param"] = ^BOOL(NSDictionary *parameters) {
  // ...
};
```

Handler Block用于参数的接收，跳转页面相关逻辑处理均在block中实现。

### 三、调用

注册完成后，可以通过如下方式进行调用，在APP内部或外部APP中都可以进行调用：

```
NSURL *viewUserURL = [NSURL URLWithString:@"myapp://user/view/joeldev"];
[[UIApplication sharedApplication] openURL:viewUserURL];
```

### 四、实例

1、注册：

```
[[JLRoutes globalRoutes] addRoute:@"/:object/:action/:primaryKey" handler:^BOOL(NSDictionary *parameters) {
  NSString *object = parameters[@"object"];
  NSString *action = parameters[@"action"];
  NSString *primaryKey = parameters[@"primaryKey"];
  // stuff
  return YES;
}];
```

2、调用：

```
NSURL *editPost = [NSURL URLWithString:@"myapp://post/edit/123?debug=true&foo=bar"];
[[UIApplication sharedApplication] openURL:editPost];
```

3、block中接收到的字典参数如下：

```
{
  "object": "post",
  "action": "edit",
  "primaryKey": "123",
  "debug": "true",
  "foo": "bar",
  "JLRouteURL": "myapp://post/edit/123?debug=true&foo=bar",
  "JLRoutePattern": "/:object/:action/:primaryKey",
  "JLRouteScheme": "JLRoutesGlobalRoutesScheme"
}
```

### 五、Schemes

以上四个步骤可以实现JLRoutes的简单使用。
JLRoutes支持自定义命名空间注册，可以对指定的URL Schemes注册路由。

```
1、[[JLRoutes globalRoutes] addRoute:@"/foo" handler:^BOOL(NSDictionary *parameters) {
  // This block is called if the scheme is not 'thing' or 'stuff' (see below)
  return YES;
}];

2、[[JLRoutes routesForScheme:@"thing"] addRoute:@"/foo" handler:^BOOL(NSDictionary *parameters) {
  // This block is called for thing://foo
  return YES;
}];
```

在此基础上注册如下路由：

```
[[JLRoutes globalRoutes] addRoute:@"/global" handler:^BOOL(NSDictionary *parameters) {
  return YES;
}];
```

如果通过 thing://global去调用，是不会拦截到的。需要做如下设置，全局路由‘/global’就会拦截到。

```
[JLRoutes routesForScheme:@"thing"].shouldFallbackToGlobalRoutes = YES;
```

### 六、Optional Routes

注册如下路由规则：

```
/the(/foo/:a)(/bar/:b)    
```

相当于注册以下四个路由：

```
/the/foo/:a/bar/:b
/the/foo/:a
/the/bar/:b
/the
```

