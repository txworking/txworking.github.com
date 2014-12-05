---
layout: post
title: "AngularJS+SpringMVC构建应用"
date: 2014-01-02 13:33
comments: true
categories: [angular, springmvc]
---

利用AngularJS和SpringMVC搭建一个简单的应用，AngularJS负责前端，SpringMVC作为后端提供REST API。

## SpringMVC的使用

SpringMVC中可以很方便的使用`@RequestMapping`注解提供REST API。

最基本的使用代码如下所示：

```
@Controller
@RequestMapping(value = "/hello")
public class MyController {

  @RequestMapping(method=RequestMethod.GET)
  @ResponseBody
  public String getRoot(){

    return "Hello World";

  }

  @RequestMapping(value="/{path}", method=RequestMethod.GET)
  @ResponseBody
  public String getValue(@PathVariable String path){
    return "Path is " + path;
  }

}
```


使用`@Controller`注解指定当前类为SpringMVC的一个controller，使用`@RequestMapping`表明此controller会处理所有以`/hello`开头的URL。

`@RequestMapping`最常用的两个参数就是`value`和`method`，分别指定了拦截的URL和请求方法。

MyController类中的两个方法还可以分别再使用`@RequestMapping(value)`再继续指定处理以hello为前缀的URL。例如`@RequestMapping(value="/{path}")`，可处理类似`/hello/path`的URL，并结合`@PathVariable`将path作为URL参数获取。

这个小Demo只是将SpringMVC作为后台REST API的提供方，也就是说只使用了MVC中的Controller和Model，View就交给AngularJS，所以使用`@ResponseBody`直接将方法返回值作为响应，不经过View的渲染啥的。

## 简单的AngularJS应用

[AngularJS](http://angularjs.org)是Google推出的前端MVC框架，也有完整的Controller、View、Model等概念，而且上手非常简单，熟悉几个html标签就能开始使用了。

建立AngularJS项目可以从GitHub上clone [angular-seed](https://github.com/angular/angular-seed)。

核心的代码都在app目录中，其典型结构如下：

*   `css` 存放css文件
*   `img` 存放图片
*   `js` 存放js代码，我们自己的代码基本都在这儿，一般按照功能和层次划分为这样几个文件：app.js, controller.js, services.js, filters.js, directives.js, animations.js。从名字就可以看出分别是在对这些概念进行定义和使用
*   `lib` 存放js库
*   `partials` 存放html模板，angularjs发送到客户端的就是这些html模板

### 基本概念

demo中涉及的几个概念：

*   app
*   controller
*   service
*   router
*   resource
*   view
*   scope

demo的基本功能就是在html模板中新建一个app和两个controller，然后自定义一个service使用REST API和SpringMVC后端交互操作hello这个resource，在controller中调用此service，html模板间的跳转通过router定义。

![Alt text](http://docs.angularjs.org/img/guide/concepts-module-service.png)

借用AngularJS官方的一张图来说明下关系：

在template中声明app，app包含controller，controller调用service，service操作resource。

层次和调用关系还是很清晰的，和Spring也差不多。

template、app、service、controller的具体代码如下

#### template
```
<!doctype html>
<html lang="en" ng-app="myApp">
<body>
  <div ng-view></div>
</body>
</html>

```

#### app
```
'use strict';

// 新建myApp，并声明会使用ngRoute、myApp.services、myApp.controllers三个module
// AngularJS会实现module的自动注入
angular.module('myApp', [
  'ngRoute',
  'myApp.services',
  'myApp.controllers'
]).
config(['$routeProvider', function($routeProvider) { // 配置app对应的URL路由，请求此URL时返回的view template和template对应的controller
  $routeProvider.when('/hello', {templateUrl: 'partials/list.html', controller: 'ListCtrl'});
  $routeProvider.when('/hello/:path', {templateUrl: 'partials/detail.html', controller: 'DetailCtrl'});
  $routeProvider.otherwise({redirectTo: '/hello'});
}]);

```
#### service
```
'use strict';

// 新建myService，并声明会使用ngResource这个module
var myService = angular.module('myApp.services', ['ngResource']).
  value('version', '0.1');
// 新建Hello这个资源，并指定资源的访问路径和访问方法
myService.factory('Hello', ['$resource',
  function($resource){
    return $resource('api/hello/:path', {}, {
      list: {method:'GET', isArray:true}
    });
  }]);

```
#### controller
```
'use strict';

// 新建ListCtrl和DetailCtrl两个controller
angular.module('myApp.controllers', []).
  controller('ListCtrl', ['$scope', 'Hello', // 声明要使用Hello
   function($scope, Hello) {
    $scope.hello = Hello.list();             // 调用Hello服务的list方法，赋值给模板中的hello元素
  }])
  .controller('DetailCtrl', ['$scope', '$routeParams', 'Hello',
   function($scope, $routeParams, Hello) {
    $scope.hello = Hello.get({path:$routeParams.path}, function(hello){
      $scope.detail = hello.detail;
    });
  }]);

```

```
<div class="container-fluid">
  <div class="row-fluid">
    <div class="span10">
      <!--Body content-->
          <p></p>
    </div>
  </div>
</div>
```

 