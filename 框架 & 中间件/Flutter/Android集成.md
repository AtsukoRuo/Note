# Android集成



## 与Android集成

https://juejin.cn/post/7054743801520193543#heading-14



这里有两个方案，（原本Android官方提供一个插件，可以很方便地进行集成，但是后面又删除了）。方案一是将Flutter打包成aar，再集成到Android。这样做有一个好处，就是Android端的开发人员无需安装Flutter SDK。

但需要注意的是，并不是修改了 `fluuter_model` 中的代码，直接重新编译 android后， 页面就会发生改变。在 android 项目中，flutter 的代码是一个 aar 包的形式存在的，所以 flutter 代码更新后，需要重新执行 `flutter build aar` 命令重新打一个aar 包才可以。而且有些插件在打包成aar后会出现问题。因此，我们更加推荐方案 B - **依赖模块的源码**。该方式可以使你的 Android 项目和 Flutter 项目能够同步一键式构建。

将 `Flutter` 模块作为子项目添加到宿主应用的 `settings.gradle` 中：

```dart
// Include the host app project.
include ':app'                                    // assumed existing content
setBinding(new Binding([gradle: this]))                                // new
evaluate(new File(                                                     // new
  settingsDir.parentFile,                                              // new
  'my_flutter/.android/include_flutter.groovy'                         // new
))                                                                     // new

```

假设 `my_flutter` 和 `MyApp` 是同级目录。

在你的应用中引入对 Flutter 模块的依赖：

```dart
dependencies {
  implementation project(':flutter')
}
```

但是上面的方法可能会报错，处理方式见https://github.com/flutter/flutter/issues/99735。

1. `settings.gradle`同级目录下，在创建`flutter_settings.gradle`文件

   ~~~dart
   setBinding(new Binding([gradle: this]))
   evaluate(new File(settingsDir.parentFile, 'flutter_module/.android/include_flutter.groovy'))
   ~~~

2. 改`settings.gradle`文件

   ~~~kotlin
   dependencyResolutionManagement {
      //repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
     repositoriesMode.set(RepositoriesMode.PREFER_SETTINGS) // edit this
       repositories {
           google()
           mavenCentral()
   }
   rootProject.name = "FlutterTestModule"
   include ':app'
   
   apply { from("flutter_settings.gradle") } // add this
   ~~~





在`Anroid`端，将`MainActivity`的父类替换为`FlutterActivity`

~~~kotlin
class MainActivity : FlutterActivity() {
    
}
~~~

有时候，会找不到`FlutterActivity`所在的包。我们对`Android`项目的`build.gradle`做如下修改

~~~
allprojects {
    repositories {
        ......
       //添加这一行
        maven { url "https://storage.googleapis.com/download.flutter.io" }
    }
}
~~~

然后，编写如下代码就可以启动一个Flutter页面了

~~~kotlin
class MainActivity : FlutterActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState);	// FlutterActivity覆写了onCreate方法
    }
}
~~~





在进行通信之前先介绍一下 `Platform Channel` ，他是 Flutter 和原生通信的工具，有三种类型：

- `BaseicMessageChannel`：用于传递字符串和半结构化信息，Flutter 和平台端进行消息数据交换时可以以使用。
- `MethodChannel` ：用于传递方法调用(method invocation)，Flutter 和平台端进行直接方法调用时候可以使用
- `EventChannel` ：用户数据流 (event stream) 的通信，Flutter 和平台端的事件监听，取消等都可以使用



`MethodChannel`作为`Flutter`端与`Android`端之间的通信对象。通过`invokeMethod()`方法来向对方发送消息。第一个参数是服务名，第二个参数是服务所携带的参数。

Flutter调用Android代码：

~~~kotlin

  MethodChannel _channel = MethodChannel("FlutterAndroid");

//调用 Android 的 AndroidMethod 方法
  var result = _channel.invokeMapMethod("AndroidMethod", "调用 Android 参数");
  result.then((value) => print('Android 返回值 :$value'));

~~~

Flutter端作为客户端，它的代码如下

```
_channel.invokeMethod("stop", "");
```

```
await _channel.invokeMethod("playMusic", {
  "name": song.name,
  "author": song.author,
  "url": song.url,
  "imgUrl": song.imgUrl,
  "ID": song.ID
});
```

Android端作为服务端，它的代码如下：

~~~kotlin
class MusicMethodChannel(messenger: BinaryMessenger): MethodChannel.MethodCallHandler {

    
    // 与Flutter通信的对象
    private val channel: MethodChannel

    init {
        // 注意通道的标识必须和Flutter保持一致
        channel = MethodChannel(messenger, "cn.AtsukoRuo")
        
        // 向Android注册接收请求时的处理方法
        channel.setMethodCallHandler(this)
    }

    override fun onMethodCall(call: MethodCall, result: MethodChannel.Result) {
        when(call.method) {
            "playMusic" -> {
   				// 获取服务所携带的参数       
                val name = call.argument("name") as String?;
                val imgUrl = call.argument("imgUrl") as String?;
                val url = call.argument("url") as String?;
                val ID = call.argument("ID") as Int?;
                val author = call.argument("author") as String?
                
                // 一定要调用success方法，否则Flutter端会报异常
                result.success(null);
            }
            "stop" -> {
                result.success(null);
            }
            "start" -> {
                // 逻辑处理
                result.success(null);
            }
            "isPlaying" -> {
                // 逻辑处理
                result.success(musicBinder?.isPlaying());
            }
        }
    }
}

~~~
