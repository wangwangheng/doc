# Retrofit

标签（空格分隔）： 第三方&开源

---

**Retrofit**是一套RESTful架构的Android(Java)客户端实现，基于注解，提供JSON to POJO(Plain Ordinary Java Object,简单Java对象)，POJO to JSON，网络请求(POST，GET,PUT，DELETE等)封装。

**相关资料：**
>[RESTful百度百科](http://baike.baidu.com/link?url=0FWilArYrZC3nMbnyIz-f7rVeANtkM_obfoIpazN9D8rQRsl4ChD2pmJLZ8aOGI3Cxvpfra0AkaPfGNGGVR3oq)
> [RESTful API设计](http://www.csdn.net/article/2013-06-13/2815744-RESTful-API)
> [Retrofit 原理](http://www.cnblogs.com/angeldevil/p/3757335.html)
> [Retrofit 简单使用](http://www.tuicool.com/articles/NnuIva)
> [Retrofit Github工程地址](https://github.com/square/retrofit.git)

Retrofit   和Java领域的ORM概念类似， ORM把结构化数据转换为Java对象，而Retrofit 把REST API返回的数据转化为Java对象方便操作。同时还封装了网络代码的调用。

例如：
```
public interface GitHubService {
    @GET("/users/{user}/repos")
    List<Repo> listRepos(@Path("user") String user);
}
```
定义上面的一个REST API接口。 该接口定义了一个函数 `listRepos`,该函数会通过HTTP GET请求去访问服务器的`/users/{user}/repos`路径并把返回的结果封装为`List<Repo>` Java对象返回。

其中URL路径中的`{user}`的值为`listRepos` 函数中的参数 `user`的取值。

然后通过  `RestAdapter`  类来生成一个  `GitHubService`  接口的实现；
```
RestAdapter restAdapter = new RestAdapter.Builder()
    .setEndpoint("https://api.github.com")
    .build();
GitHubService service = restAdapter.create(GitHubService.class);
```
获取接口的实现后就可以调用接口函数来和服务器交互了；
```
List<Repo> repos = service.listRepos("octocat");
```
-------

从上面的示例可以看出， Retrofit 使用注解来声明HTTP请求

* 支持 URL 参数替换和查询参数
* 返回结果转换为Java对象(返回结果可以为JSON, protocol buffers)
* 支持 Multipart请求和文件上传

##具体使用文档

函数和函数参数上的注解声明了请求方式

### 1、请求方式

每个函数都必须带有 HTTP 注解来表明请求方式和请求的URL路径。类库中有5个HTTP注解:**GET**, **POST**, **PUT**,**DELETE**和**HEAD**。注解中的参数为请求的**相对URL路径**。

```
@GET("/users/list")
```

在URL路径中也可以指定URL参数
```
@GET("/users/list?sort=desc")
```

### 2、URL处理

请求的URL可以根据函数参数动态更新。一个可替换的区块为用 `{` 和 `}` 包围的字符串，而函数参数必需用  `@Path`注解标明，并且*注解的参数为* **同样的字符串**

```
@GET("/group/{id}/users") //注意 字符串id
List<User> groupList(@Path("id") int groupId); //注意 Path注解的参数要和前面的字符串一样 id
```

还支持查询参数
```
@GET("/group/{id}/users")
List<User> groupList(@Path("id") int groupId, @Query("sort") String sort);
```

### 3、请求体（Request Body）

通过`@Body`注解可以声明一个对象作为请求体发送到服务器。
```
@POST("/users/new")
void createUser(@Body User user, Callback<User> cb);
```
对象将被`RestAdapter`使用对应的转换器转换为字符串或者字节流提交到服务器。

### 4、FORM ENCODED AND MULTIPART 表单和Multipart

函数也可以注解为发送表单数据和multipart 数据

使用`@FormUrlEncoded`注解来发送表单数据；使用`@Field` 注解和参数来指定每个表单项的Key，value为参数的值。
```
@FormUrlEncoded
@POST("/user/edit")
User updateUser(@Field("first_name") String first, @Field("last_name") String last);
```
使用`@Multipart`注解来发送`multipart`数据。使用`@Part`注解定义要发送的每个文件。
```
@Multipart
@PUT("/user/photo")
User updateUser(@Part("photo") TypedFile photo, @Part("description") TypedString description);
```
Multipart 中的Part使用`RestAdapter`的转换器来转换，也可以实现`TypedOutput`来自己处理序列化。

### 5、异步 VS 同步

每个函数可以定义为异步或者同步。

#### 5.1、具有返回值的函数为同步执行的

```
@GET("/user/{id}/photo")
Photo listUsers(@Path("id") int id);
```

#### 5.2、异步执行函数没有返回值并且要求函数最后一个参数为Callback对象

```
@GET("/user/{id}/photo")
void listUsers(@Path("id") int id, Callback<Photo> cb);
```
> * 在 Android 上，callback对象会在主(UI)线程中调用。
> * 在普通Java应用中，callback在请求执行的线程中调用。

### 6、服务器结果转换为Java对象

使用RestAdapter的转换器把HTTP请求结果（默认为JSON）转换为Java对象，Java对象通过函数返回值或者Callback接口指定
```
// 通过返回值指定要转换的Java的对象的类型
@GET("/users/list")
List<User> userList();
```

```
// 通过Callback的泛型参数来制定要转换的Java对象的类型
@GET("/users/list")
void userList(Callback<List<User>> cb);
```

### 7、如果要直接获取HTTP返回的对象，使用Response对象。

```
// 通过返回值的方式指定
@GET("/users/list")
Response userList();
```

```
// 通过Callback的泛型参数指定
@GET("/users/list")
void userList(Callback<Response> cb);
```

### 8、Retrofit涉及的重要知识点

* 动态代理
* 注解
* Handler + MessageQueue + Thread + Looper机制
* OKHttp请求网络
* GSON用来转换Java POJO和解析JSON