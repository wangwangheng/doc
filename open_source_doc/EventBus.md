# EventBus

标签（空格分隔）： 第三方&开源

---

[TOC]


![EventBus](https://github.com/greenrobot/EventBus/raw/master/EventBus-Publish-Subscribe.png)

[Github项目官方网站](https://github.com/greenrobot/EventBus)
Git仓库地址：[https://github.com/greenrobot/EventBus.git](https://github.com/greenrobot/EventBus.git)

## 1、EventBus的使用步骤

### 1.1.定义事件

EventBus的事件是一个POJO(plain old Java object,简单Java对象)，没有任何特定的要求

```
public class MessageEvent {
    public final String message;

    public MessageEvent(String message) {
        this.message = message;
    }
}
```


### 1.2.准备订阅者

订阅者实现一个`onEvent`方法，当事件被发送出来的时候，这个方法将会被调用。它还需要注册和注销事件。

```
 @Override
public void onCreate() {
    super.onStart();
    EventBus.getDefault().register(this);
}

@Override
public void onDestory() {
    EventBus.getDefault().unregister(this);
    super.onStop();
}

// 当发布者发送的事件类型是MessageEvent的时候将会被调用
public void onEvent(MessageEvent event){
    Toast.makeText(getActivity(), event.message, Toast.LENGTH_SHORT).show();
}

// 当发布者发送的事件类型是SomeOhterEvent的时候将会被调用
public void onEvent(SomeOtherEvent event){
    doSomethingWith(event);
}
```


### 1.3.发布事件

在你的代码的任何地方都可以发送任意类型的事件，任何和事件匹配的对象将会接收到这个事件。
```
 EventBus.getDefault().post(new MessageEvent("Hello everyone!"));
```

## 2.发送线程和线程模型

你可以在和发布者不同的线程中处理事件。一个常见的用例是处理UI的变化,在Android中，改变UI必须在主线程中进行，另外一方面，网络请求和其他耗时的任务都**绝对不能**在主线程中执行。EventBus帮助你处理这些线程和UI线程的同步。无需深究线程转换、使用AsyncTask等。

在EventBus中，你可以通过`ThreadMode`定义处理方法所在的线程:

### 2.1 PostThread

订阅者的方法将在和发布事件(发布者)的线程中调用。这是EventBus的默认Thread Mode.这种事件传递方式消耗最少，因为它避免了线程切换的开销。针对比较小的任务，这个是一种推荐的模式。事件处理函数应该快速处理完任务并返回，以免阻塞事件发送线程，因为事件发送线程可能是主线程。例子：

```
// Called in the same thread (default)
public void onEvent(MessageEvent event) {
    log(event.message);
}
```

### 2.2 MainThread

订阅者的方法将在主线程中被调用(UI线程)，如果发布者发布事件线程是主线程，订阅者的方法将被直接调用。订阅者在这个方法中应该尽快返回，以免阻塞主线程：

```
// Called in Android UI's main thread
public void onEventMainThread(MessageEvent event) {
    textField.setText(event.message);
}
```

### 2.3 BackgroundThread

订阅者的方法将在一个后台线程被调用，如果发布者发布事件不在主线程中，那么订阅者的方法将会被直接调用(此时和PostThread模式一样)；如果发布者在主线程中发布事件，则它会在意唯一的后台线程中按照序列的方式调用订阅者的处理函数。这个模式的处理函数应该尽快返回避免阻塞后台线程。

```
// Called in the background thread
public void onEventBackgroundThread(MessageEvent event){
    saveToDisk(event.message);
}
```

### 2.4 AsyncThread

订阅者的方法在一个单独的线程被调用。这是独立于发布线程和主线程的。发布者不会等待时间处理函数处理完，当事件处理函数需要执行比较耗时的操作的时候，可以使用这种模式，比如网络请求。避免大量使用长时间运行的异步事件处理方法，同时要限制并发线程数。eventbus使用线程池来有效地重用线程。

```
 // Called in a separate thread
public void onEventAsync(MessageEvent event){
    backend.send(event.message);
}
```


## 3. 订阅者优先级和有序的事件传递

可以在register订阅者的时候指定一个优先级参数来指定事件传递的顺序。

```
int priority = 1;
EventBus.getDefault().register(this, priority);
```
在同一种线程模式中，高优先级的订阅者比低优先级的订阅者优先处理事件。优先级的默认值是0.

> 特别需要注意的是，优先级不会影响不同线程模式的方法调用

## 4. 使用EventBusBuilder配置EventBus

EventBus2.3版本增加了`EventBusBuilder`来配置EventBus的各项参数。例如，下面展示了在没有订阅者的时候发布一个事件的情况：
```
EventBus eventBus = EventBus.builder()
	.logNoSubscriberMessages(false)
	.sendNoSubscriberEvent(false)
	.build();
```

另一个例子是当订阅者事件处理函数抛出一个异常的时候EventBus的处理方式，需要说明的是，默认情况下，EventBus会捕捉订阅者的事件处理函数并发出一个`SubscriberExceptionEvent`事件,而且订阅者可以完全不需要关心：
```
EventBus eventBus = EventBus.builder().throwSubscriberException(true).build();
```
检查EventBusBuilder类和java doc 可以查看所有的可配置选项。

## 5、配置默认的EventBus实例

使用`EventBus.getDefault()`是使用EventBus的共享实例的一种简单的方式。`EventBusBuilder`也允许使用`installDefaultEventBus()`方法配置EventBus的默认实例。

例如，可以配置默认EventBus实例在onEvent方法抛出异常的时候来重新抛出异常，我们可以只针对Debug版本进行这样的配置：
```
EventBus.builder().throwSubscriberException(BuildConfig.DEBUG).installDefaultEventBus();
```
> 注意：这个配置应该在第一次使用EventBus之前使用，以确保你的应用程序的行为一致性。一般是在Application类中进行配置比较好。

## 6、取消事件传递

你可以在订阅者的事件处理方法中调用`cancelEventDelivery(Object event)`函数来取消事件的传递。任何进一步的事件传递将取消，后续的订阅者不会接收事件：
```
// Called in the same thread (default)
public void onEvent(MessageEvent event){
    // Process the event 
    ...

    EventBus.getDefault().cancelEventDelivery(event) ;
}
```

取消事件的传递一般都是在高优先级的订阅者中进行，取消传递的执行线程是和发布事件的线程在同一个线程中。

## 7、粘性事件

一些事件携带的信息是事件发布后订阅者感兴趣的，例如，可能是一个初始化完成的事件信号或者如果你有一些传感器或位置数据和你要保持最新的值需要传递，你可以用粘性事件，而不是实现自己的缓存。EventBus在内存中维持一个最新的事件。粘性事件可被传递给订阅者，或者通过明确的方法调用得到。因此，你不需要任何特殊的逻辑考虑已经可用的数据。

比方说，一个粘性事件已经发布了一段时间：

```
EventBus.getDefault().postSticky(new MessageEvent("Hello everyone!"));
```
然后一个Activity被启动，如果注册的时候使用`registerSticky()`方法，则可以得到之前发布的粘性事件：
```
@Override
public void onStart() {
    super.onStart();
    EventBus.getDefault().registerSticky(this);
}

public void onEventMainThread(MessageEvent event) {
    textField.setText(event.message);
}

@Override
public void onStop() {
    EventBus.getDefault().unregister(this);
    super.onStop();
}
```
你也可调用方法得到这个事件：
```
EventBus.getDefault().getStickyEvent(Class<?> eventType)

```
也可以通过`removeStickyEvent()`方法删除一个粘性事件。EventBus最多只会保留一个类型的粘性事件。


## 8、Proguard配置

事件处理的方法名称不能被混淆，如果代码需要混淆，可以在proguard.cfg文件中添加以下的配置来忽略混淆EventBus:

```
-keepclassmembers class ** {
    public void onEvent*(**);
}

# Only required if you use AsyncExecutor
-keepclassmembers class * extends de.greenrobot.event.util.ThrowableFailureEvent {
    <init>(java.lang.Throwable);
}
```

## 9、其他内容

AsyncExecutor和AsyncExecutor Builder可以参考[官方文档](https://github.com/greenrobot/EventBus/blob/master/HOWTO.md)

和Square的OTTO开源项目的对比可以参考[对比文档](https://github.com/greenrobot/EventBus/blob/master/COMPARISON.md)