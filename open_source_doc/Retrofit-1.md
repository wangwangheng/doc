# Retrofit文档

标签（空格分隔）： 第三方&开源

---

[TOC]

## 1. 入门

* Retrofit转换你的`REST API`到一个Java接口：
```
public interface GitHubService {
  @GET("/users/{user}/repos")
  List<Repo> listRepos(@Path("user") String user);
}
```

* `RestAdapter`生成`GitHubService`接口的一个实现：
```
RestAdapter restAdapter = new RestAdapter.Builder()
    .setEndpoint("https://api.github.com")
    .build();

GitHubService service = restAdapter.create(GitHubService.class);
```
* 每次调用`GitHubService`的一个方法都会对远程Web服务器发出一个HTTP请求：
```
List<Repo> repos = service.listRepos("octocat");
```
* Retrofit使用注解描述HTTP
* 对象转换为请求体(例如JSON，协议缓冲区)
* Multipart请求体和文件上传

-----

## 2. API说明

* 在接口的方法和它的参数添加注解来说明请求如何被处理。

### 2.1 请求方法

* 每一个方法必须有一个HTTP注解，用来提供请求方法和相对URL。有五个内置注解：`GET`,`POST`,`PUT`,`DELETE`,`HEAD`。资源的相对URL被指定在这个注解中。
```
@GET("/users/list")
```

* 你也可以在相对URL中指定查询参数
```
@GET("/users/list?sort=desc")
```

### 2.2 URL处理

* 一个URL可以通过替换块和方法的参数动态的被更新。一个替换块是一个被`{`和`}`包围的字母数字字符串。对应的参数必须在`@Path`注解中使用相同的字符串:
```
@GET("/group/{id}/users")
List<User> groupList(@Path("id") int groupId);
```

* 也可以添加查询参数:
```
@GET("/group/{id}/users")
List<User> groupList(@Path("id") int groupId, @Query("sort") String sort);
```

* 对于复杂的查询参数可以使用组合的Map：
```
@GET("/group/{id}/users")
List<User> groupList(@Path("id") int groupId, @QueryMap Map<String, String> options);
```

### 2.3 请求体

* 一个对象可以通过`@Body`注解指定为HTTP请求体：
```
@POST("/users/new")
void createUser(@Body User user, Callback<User> cb);
```
* 这个对象也可以被`RestAdapter`的转换器转换。

### 2.4 表单格式和Multipart

* 方法也可以被声明为用来发送表单格式的数据和multipart(主要用来文件上传)数据。
* 当`@FormUrlEncoded `注解存在于方法上的时候，数据将以表单格式发送。每个`键 -值对`的键是 `@Field`的`value`,值是`@Field`所注解的参数的值：
```
@FormUrlEncoded
@POST("/user/edit")
User updateUser(@Field("first_name") String first, @Field("last_name") String last);
```

* 当使用一个`@Multipart`注解一个方法的时候，数据将以Multipart的格式发送，使用`@Part`来注解每一个Part:
```
@Multipart
@PUT("/user/photo")
User updateUser(@Part("photo") TypedFile photo, @Part("description") TypedString description);
```
* Multipart的parts使用`RestAdapter`的转换器或者它们实现`TypeOutput`去处理它们自己的序列化

### 2.5 处理Header

* 你能够使用`@Header`注解来设置静态的header信息：
```
@Headers("Cache-Control: max-age=640000")
@GET("/widget/list")
List<Widget> widgetList();
```

```
@Headers({
    "Accept: application/vnd.github.v3.full+json",
    "User-Agent: Retrofit-Sample-App"
})
@GET("/users/{username}")
User getUser(@Path("username") String username);
```

* 需要注意的是**Header不会相互覆盖**。所有的header将以相同的名称包含在请求里面。

* 请求的Header可以使用`@Header`注解进行动态的更新。一个符合条件的参数必须使用`@Header`来提供，如果你提供的参数值为null,则这个Header将被忽略，否则将会使用这个参数的`toString()`函数的调用结果：
```
@GET("/user")
void getUser(@Header("Authorization") String authorization, Callback<User> callback)
```

* 如果需要在每一个请求中添加相同的Header，可以使用`RequestInterceptor`,下面的代码创建一个`RequestInterceptor`为每一个请求添加一个`User-Agent`的请求头：
```
RequestInterceptor requestInterceptor = new RequestInterceptor() {
  @Override
  public void intercept(RequestFacade request) {
    request.addHeader("User-Agent", "Retrofit-Sample-App");
  }
};

RestAdapter restAdapter = new RestAdapter.Builder()
  .setEndpoint("https://api.github.com")
  .setRequestInterceptor(requestInterceptor)
  .build();
```

### 2.6 同步、异步和Observable

方法能够被声明为同步或者异步执行。

* 如果一个方法有返回值，则它将同步的执行:
```
@GET("/user/{id}/photo")
Photo getUserPhoto(@Path("id") int id);
```

* 异步的执行一个方法需要方法的最后一个参数是一个`Callback`
```
@GET("/user/{id}/photo")
void getUserPhoto(@Path("id") int id, Callback<Photo> cb);
```
在Android中，callback在主线程中执行。对于桌面应用程序callback将会在执行Http请求的线程中执行。

* Retrofit也集成了[RxJava](https://github.com/ReactiveX/RxJava/wiki)，可以让方法返回`rx.Observable`:
```
@GET("/user/{id}/photo")
Observable<Photo> getUserPhoto(@Path("id") int id);
```
Observable请求是订阅异步请求并且在执行HTTP请求的相同线程上观察，调用在不同的线程的（例如Android的主线程）observeOn(Scheduler)方法对返回的Observable进行观察。

### 2.7 响应对象类型
* `RestAdapter`的转换器会自动把HTTP的响应数据转换为指定的类型(默认是JSON)，想要的类型通过方法的返回值类型，Callback的泛型类型或者Observable的泛型类型来指定：
```
@GET("/users/list")
List<User> userList();

@GET("/users/list")
void userList(Callback<List<User>> cb);

@GET("/users/list")
Observable<List<User>> userList();
```

* 使用`Response`类型来访问HTTP请求的原生响应结果：
```
@GET("/users/list")
Response userList();

@GET("/users/list")
void userList(Callback<Response> cb);

@GET("/users/list")
Observable<Response> userList();
```
------

## 3. RestAdapter配置

`RestAdapter`是把你的API接口转换为可调用的方法的类(实际上是通过动态代理来生成具体的请求类的对象)，默认情况下Retrofit为合适的平台提供合适的默认配置，但是它也允许自定义配置。

### 3.1 JSON转换器
Retrofit默认使用[Gson](https://code.google.com/p/google-gson/)来转换请求参数并把服务端返回的原生数据转换为你指定的类型。如果你想指定不同的Gson的默认行为(如命名策略，数据类型，自定义类型)，可以在build一个`RestAdapter`的时候指定一个包含你所期望行为新的Gson的实例，可以参考[Gson文档](https://sites.google.com/site/gson/gson-user-guide)来了解自定义Gson行为。

### 3.2 自定义JSON转换器的实例
下面的代码创建一个把所有字段从用下划线分割的小写字母转换为驼峰命名法。它也为`Date`类注册了一个类型的适配器。这个`DateTypeAdapter`将会在Gson解析日期类型的时候使用。

该`gson`实例作为`GsonConverter`参数，这是一个用于转换类型的包装类。

```
Gson gson = new GsonBuilder()
    .setFieldNamingPolicy(FieldNamingPolicy.LOWER_CASE_WITH_UNDERSCORES)
    .registerTypeAdapter(Date.class, new DateTypeAdapter())
    .create();

RestAdapter restAdapter = new RestAdapter.Builder()
    .setEndpoint("https://api.github.com")
    .setConverter(new GsonConverter(gson))
    .build();

GitHubService service = restAdapter.create(GitHubService.class);
```

* 调用每个`GithubServic`的实现将返回数据将被设置到`RestAdapter`的GSON对象转换为需要的对象。

### 3.3 其他的转换器

除了JSON，Retrofit也可以配置其他的内容格式，Retrofit提供了可选的XML转换器(使用[Sample][Sample] -- 一个高性能的XML序列化和配置框架)和协议缓冲池(使用[protobuf][protobuf] 或者 [Wire][Wire]).请查看[retrofit-converters][retrofit-converters]目录查看所有的转换器列表。

下面的代码展示了怎样使用`SimpleXMLConverter `与使用XML的API通讯：

```
RestAdapter restAdapter = new RestAdapter.Builder()
    .setEndpoint("https://api.soundcloud.com")
    .setConverter(new SimpleXMLConverter())
    .build();

SoundCloudService service = restAdapter.create(SoundCloudService.class);
```

[Sample]:http://simple.sourceforge.net/
[protobuf]:https://code.google.com/p/protobuf/
[Wire]:https://github.com/square/wire
[retrofit-converters]:https://github.com/square/retrofit/tree/master/retrofit-converters


### 3.4 自定义转换器

入股哦你需要使用一个Retrofit不支持的内容格式与API通讯(如YAML，txt,自定义格式)或者或者你想使用一个不同而库来实现现有格式，你可以轻松的创建你自己的转换器。创建一个类并且实现[`Converter`接口][converter-interface]并且在创建适配器的时候传递一个实例。

[converter-interface]:https://github.com/square/retrofit/blob/master/retrofit/src/main/java/retrofit/converter/Converter.java

### 3.5 自定义错误处理器
如果你需要自定义错误处理的请求，您可以提供自己的`ErrorHandler`。下面的代码展示了怎样在服务器返回401状态码的时候抛出一个自定义的异常：
```
class MyErrorHandler implements ErrorHandler {
  @Override 
  public Throwable handleError(RetrofitError cause) {
    Response r = cause.getResponse();
    if (r != null && r.getStatus() == 401) {
      return new UnauthorizedException(cause);
    }
    return cause;
  }
}

RestAdapter restAdapter = new RestAdapter.Builder()
    .setEndpoint("https://api.github.com")
    .setErrorHandler(new MyErrorHandler())
    .build();
```

注意，如果返回的异常是被封闭的，它必须被定义在接口方法中。建议您通过我们提供的RetrofitError的错误原因抛出新的异常。

### 3.6 日志

如果你需要详细查看请求和响应，你可以很轻松地添加日志级别到`RestAdapter`中。可能的日志级别是:`BASIC`,`FULL`,`HEADERS`和`NONE`.

下面的代码展示了如何记录Header，Boady，和元数据请求和响应的一个完整的日志级别：

```
RestAdapter restAdapter = new RestAdapter.Builder()
    .setLogLevel(RestAdapter.LogLevel.FULL)
    .setEndpoint("https://api.github.com")
    .build();
```
这个日志能够在`RestAdapter`的任何生命周期的时间点通过调用`.setLogLevel()`方法添加或者改变不同的日志级别的值。

* [下载jar包](http://repository.sonatype.org/service/local/artifact/maven/redirect?r=central-proxy&g=com.squareup.retrofit&a=retrofit&v=LATEST)

* [Github代码下载](https://github.com/square/retrofit)

* 全文参考：http://square.github.io/retrofit/#download
* 没有加入Gradle、Maven相关的内容。