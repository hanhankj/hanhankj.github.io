# **Base** 是一个用于Android快速开发框架，主要包含：

### 网络请求

NetApi.httpClient网络请求对象，可被自定义OkHttpClient对象。当不设置的NetApi.httpClient的时候默认采用以下配置：

```kotlin
OkHttpClient.Builder().apply {
    protocols(listOf(Protocol.HTTP_1_1, Protocol.HTTP_2))
    connectTimeout(timeout, TimeUnit.SECONDS)
    writeTimeout(timeout, TimeUnit.SECONDS)
    readTimeout(timeout, TimeUnit.SECONDS)
    addInterceptor(HttpLoggingInterceptor().apply {
        level = HttpLoggingInterceptor.Level.BODY
    })
}
```

timeout可以通过NetApi.timeout配置; 网络全局错误处理： defApiCatch:(Throwable) -> Boolean

```kotlin
//默认的全局错误处理
defApiCatch = {
    when (it) {
        is SocketTimeoutException -> {
            showToast("网络连接超时")
            true
        }
        is ErrnoException, is UnknownHostException, is SocketException, is ConnectException -> {
            showToast("网络异常,请检查网络")
            true
        }
        is NetException -> {
            (it.msg ?: "服务异常").toast(Toast.LENGTH_LONG)
            true
        }
        else -> false
    }
}
```

全局错误处理可以被继承：

```kotlin
val globalApiCatch = defApiCatch
defApiCatch = {
    if (globalApiCatch(it).not()) {
        when (it) {
            is HttpException -> {
                showToast("${it.code()}->${it.message()}")
                true
            }
            else -> false
        }
    } else {
        false
    }
}
```

定义网络请求结果解析规则,需要继承IApiResult<R>：

| 字段     | 解释          |
|-----|-------------|
| isSuccessful | 定义该请求是否成功   |
|   apiCode     | 当前请求的code   |
|   apiData     | 当前请求结果      |
|   apiMsg     | 当前请求结果的状态信息 |

例如：

```kotlin
data class TestApiResult<R>(
    var success: Boolean,
    var code: Int,
    var result: R?,
    var message: String?,
) : IApiResult<R> {
    override val isSuccessful: Boolean get() = success
    override val apiCode: Int get() = if (success) 200 else code
    override val apiData: R? get() = result
    override val apiMsg: String get() = message ?: "网络异常"
}
```

网络API定义，具体配置请参考retrofit,例如：

```kotlin
interface TestApi {
    @GET("api/token/captcha/sms")
    suspend fun getVerifyCode(): TestApiResult<String>
}
```

API的配置方法（baseUrl="http://192.168.1.100:8080/test")：

| 方法                            | 说明                                                                   | 示例                         |
|-------------------------------|----------------------------------------------------------------------|----------------------------|
| addApi&lt;T>(baseUrl: String) | 注册api,该方法等同于<br/>NetApi.registerService(baseUrl: String, clazz: Class&lt;T>) | addApi&lt;TestApi>(baseUrl) |
| delApi&lt;T>()                | 移除api 该方法等同于<br/>removeService(clazz: Class&lt;T>)                                                          | delApi&lt;TestApi>()       |
| api&lt;T>()                   | 获取api 该方法等同于<br/>NetApi.getService(clazz: Class&lt;T>)                                                         | api&lt;TestApi>            |

由于该网络请求定义都是基于Kotlin**协程**开发的，所以发起网络请求需要配合协程，例如：

```kotlin
launch {
    showLoading()
    val iApiResult = srcData { api<TestApi>().getVerifyCode() }
    dismissLoading()
    if (iApiResult == null) {
        Log.e("net", "网络请求失败")
        return@launch
    }
    //网络结果处理方式一
    if (iApiResult.isSuccessful) {
        iApiResult.apiData!!.toast(Toast.LENGTH_LONG)
    } else {
        "${iApiResult.apiMsg}->个性化".toast(Toast.LENGTH_LONG)
    }
    //网络结果处理方式二
    //该方式直接获取结果，如果apiData为空，根据配置提醒错误和处理相关的错误信息
    val result = iApiResult.data { it.toast() } ?: return@launch
    result.toast()
}
```

也可以这样发起请求：

```kotlin
launch {
    showLoading()
    val result = data { api<TestApi>().getVerifyCode() }
    dismissLoading()
    Log.e("net", "网络请求结束")
    result ?: return@launch
    result.toast(Toast.LENGTH_LONG)
}
```

推荐该方式，更简便，该方式可以处理特定的错误，但更推荐在全局错误，除非是需要根据特定的错误完成特定的动作。

### 缓存管理（键值对）

CacheManage是一个缓存管理器，在使用之前需要调用```CacheManager.init(context:Application,processMode:CacheProcessMode)```
初始化，
该管理器主要提供了```put(key:String,value:Any)```和```get***(key:String)```方法, 比如：

```
CacheManage.put("demoKey","demoValue")
val demoValue = getString("demoKey")
```

| 方法名                                       | 说明                                          | 示例                                      |
|-------------------------------------------|---------------------------------------------|-----------------------------------------|
| init(context:Application,processMode:Int) | 初始化，建议在Application中调用 | CacheManager.put("demoKey","demoValue") |
| put(key:String,value:Any)                 | 添加缓存                                        | CacheManager.put("demoKey","demoValue") |
| get***(key:String):***?                   | 查询缓存,***可以是：String,Int,Float,Double,Boolean | CacheManager.getString("demoKey")       |
| get<T>(key:String):T?                     | 查询缓存,使用泛型                                   | CacheManager.get<Size>("putAny")        |
| getList<T>(key:String):List<T>?           | 查询列表,使用泛型                                   | CacheManager.getList<Size>("putAny")    |
| getMap<T>(key:String):Map<T>?             | 查询Map,使用泛型                                  | CacheManager.getMap<Size>("putAny")     |
| getAllKeys(key:String):String[]           | 查询所有的缓存Key                                  |                                         |
| containsKey(key:String):Boolean           | 判断缓存Key 是否存在                                |                                         |
| remove(vararg keys: String)               | 批量移除缓存                                      |                                         |

### 权限管理

```kotlin
val permissions = arrayOf(
    Manifest.permission.CAMERA,
    Manifest.permission.RECORD_AUDIO,
    Manifest.permission.WRITE_EXTERNAL_STORAGE,
    Manifest.permission.CALL_PHONE,
    Manifest.permission.ACCESS_FINE_LOCATION,
    Manifest.permission.MODIFY_AUDIO_SETTINGS,
)
//判断是否已经拥有权限
hasPermissions(permissions)
//申请权限
easyPermission(
    permissions,
    true,
    permissionConfig = PermissionConfig(
        goSettingAlertTitleAndMsg = AlertTitleAndMsg(
            "授权提醒",
            "去系统设置页授权？"
        ),
        goSettingButton = DialogButton("去授权", "我再想想")
    )
) { it: Map<String, Boolean> ->
    it.forEach {
        Log.e("permission", "权限${it.key}${if (it.value) "被授权" else "被拒绝"}")
    }
}
```

### Web加载

Web支持加载现在H5、离线h5 zip包。该功能的入口类都是**WebManager**

|方法/字段| 说明                                          | 示例                                                    |
|----|---------------------------------------------|-------------------------------------------------------|
|  gpsWhitelistPage   | h5使用gps的白名单地址，只要有权限就允许该地址的h5使用gps           | WebManager.gpsWhitelistPage.add("map.baidu.com")      |
|   blacklistJumpIntentPage  | WebView是支持打开隐式Intent的，可以配置该字段可以禁止h5打开Intent | WebManager.blacklistJumpIntentPage.add("*.baidu.com") |
| openWebApp(activity: FragmentActivity,config: WebAppConfig,activityResultCallback: ResultCallback)    | 打开h5                                        | 见下面示例                                                 |
|  openMiniApp(activity: AppCompatActivity,miniAppConfig: MiniAppConfig,activityResultCallback: ResultCallback,openCallback: ((Pair<Boolean, Exception?>) -> Unit)?)   | 打开离线h5                                      | 见下面示例                                                 |
|   registerApi(moduleName: String, api: BaseWebExtApi)  | 注册Web与原生沟通的Api桥梁模块                          |                                                       |
|   removeApi(moduleName: String)   | 移除Web与原生沟通的Api                              |                                                       |

Web界面的标题栏配置，通过TitleConfig与WebAppConfig实现，可配置项：

**TitleConfig**

| 字段名/方法                                                                | 说明                                              |示例|
|-----------------------------------------------------------------------|-------------------------------------------------|----|
| backgroundColor：Int                                                   | 背景色，默认Color.WHITE                               |  |
| titleName: String                                                     | 在未加载完成时的标题名称，加载完成后按h5的标题显示                      |  |
| titleNameColor: Int                                                   | 标题的字体颜色，默认Color.BLACK                           |  |
| dividerVisible: Int                                                   | 标题栏与内容直接的分割线是否显示，默认View.VISIBLE                 |  |
| leftImg: String                                                       | 标题栏左侧图标的资源描述，支持Url,”R.drawable.**“,Int.toString |  |
| leftImgStartMargin: Int                                               | 左侧图标Start边距                                     |  |
| leftImgListenerKey: String                                            | 左侧图标Click事件Key                                  |  |
| rightImg: String                                                      | 标题栏右侧图标的资源描述，支持Url,”R.drawable.**“,Int.toString |  |
| rightImgEndMargin: Int                                                | 右侧图标Start边距                                     |  |
| rightImgListenerKey: String                                           | 右侧图标Click事件Key                                  |  |
| titleClickListener:hashMapOf<String, (activity: Activity) -> Unit>()  | Click事件设置                                       |  |

**WebAppConfig**

| 字段名/方法               | 说明                       |示例|
|----------------------|--------------------------|----|
| titleConfig          | 标题栏配置                    |  |
| statusBarColor:Int   | 状态栏颜色，默认Color.WHITE      |  |
| navigationBarColor   | 导航栏颜色, 默认Color.WHITE     |  |
| windowLightStatusBar | 状态栏是否是亮色模式，默认true,影响状态栏字体颜色 |  |
| mergeStatusBar       | 使用使用Edge to Edge，默认false |  |

Api需要继承BaseWebExtApi,定义有两种方式，推荐使用方式一：

> 方式一，不需要重写invoke方法：
> > 同步方法定义:methodName(fragment: Fragment, params: String):String
>
> > 异步方法定义:methodName(fragment: Fragment, params: String, callback: ICallback)
>
> 方式二，重写invoke方法

方式一定义示例:

```kotlin
class TestWebApi : BaseWebExtApi {
    //同步方法,
    fun toast(fragment: WebFragment, params: String): String {
        showToast(params)
        return "native received $params"
    }

    //异步方法
    fun getBaiduHtml(fragment: WebFragment, params: String, callback: ICallback) {
        fragment.launch {
            try {
                val bdHtml = launchIO {
                    "https://www.baidu.com/index.html".urlToTxt()
                }
                callback.onSuccess(bdHtml!!.toString())
            } catch (e: Exception) {
                callback.onFail(e.message ?: "发生异常")
            }
        }
    }
}
```

注册示例：

```kotlin
/**
 *  fun invoke(fragment: WebFragment, method: String, params: String): String
 *  fun invoke(fragment: WebFragment, method: String, params: String, callback: ICallback)
 */
WebManager.registerApi("reflex", TestWebApi())
```

打开H5打开示例：

```kotlin
val rightKey = System.currentTimeMillis().toString()
titleClickListener[rightKey] = {
    AlertDialog.Builder(it)
        .setMessage("当前正在访问百度")
        .show()
}
val leftKey = System.currentTimeMillis().toString() + "_left"
titleClickListener[leftKey] = {
    it.onBackPressed()
}
WebAppConfig("https://www.baidu.com/").apply {
    titleConfig = TitleConfig(
        titleName = "百度一下",
        backgroundColor = ContextCompat.getColor(
            this@MainActivity,
            R.color.primary_color
        ),
        titleNameColor = Color.WHITE,
//        leftImgStartMargin = 15.dp,
        leftImg = "R.drawable.title_back_img",
//        leftImg = "https://www.baidu.com/img/flexible/logo/pc/result.png",

        dividerVisible = View.GONE,

        rightImg = "https://www.baidu.com/img/flexible/logo/pc/result.png",
        rightImgEndMargin = 15.dp,
        rightImgListenerKey = rightKey
    )
    statusBarColor = ContextCompat.getColor(this@MainActivity, R.color.primary_color)
    windowLightStatusBar = false
    mergeStatusBar = false
}
```

H5调用原生方法：

调用baseApi的showToast(示例)同步方法，

```html
const toastParams = JSON.stringify({
content:"这是一个来自h5的toast",
duration:1
})
window.invokeSync("baseApi","showToast",toastParams)
```

调用baseApi的openFile(示例)异步方法,

```html
const params = JSON.stringify({
filePath:"",
mimeType:""
})
window.invoke("baseApi","openFile",params).then(res=>{}).catch((err)=>{})
```

SDK提供的H5基础方法包括：

模块名：baseApi

| 方法名 | 说明           | 参数                                                            | 备注                                                                              |
| --- |--------------|---------------------------------------------------------------|---------------------------------------------------------------------------------|
| showToast | 同步方法，显示toast | ``` { content:"content","druation":1 }```                     |                                                                                 |
| previewMedia | 同步方法，预览图片或视频 | ``` { paths: [{ path:"",mimeType:"" },currentPosition:0] }``` |                                                                                 |
| playVideo | 同步方法，播放视频    | ``` { urls:["url1","url2"]}```                                |                                                                                 |
| openFile | 异步方法，打开文件    | ``` { filePath:"",mimeType:""}```                             |                                                                                 |
| printLog | 同步方法，打印日志    | ``` { tag:"",log:"", level: 1}```                             | level取值：1->v,2->d,e->i,4->w,5->e；v、d、i、w、e指的是Android的Log级别                      |
| getStatusBarHeight | 同步方法，获取状态栏高度 | ""                                                            |                                                                                 |
| saveFile    | 异步方法，保存文件    | ```{filePath:"",fileName:"",storeType:"private"}```            | storeType取值：private,public;private:下载到私有目录(会覆盖文件名相同的文件)，public:下载到公有Downloads目录 |

模块名: appInfoApi

| 方法名 | 说明             | 参数 | 备注                                            |
| --- |----------------|---|-----------------------------------------------|
| getVersionInfo    | 同步方法，获取app版本信息 | ""  | 返回参数：```{versionCode:1,versionName:"1.0.1"}``` |

模块名: controlApi

| 方法名 | 说明        | 参数                                    | 备注                   |
| --- |-----------|---------------------------------------|----------------------|
| setStatusBarColor | 同步方法，设置状态栏颜色 | ```{color:"#FFFFFF"}```               |                      |
| showLoading | 同步方法，显示loading  | ```{content:"加载中"}```                 |                      |
| hideLoading | 同步方法，隐藏loading  | ""                                    |                      |
| closeWindow | 同步方法，关闭当前Web | "" 或 ```{resultCode:0,resultData:""}``` | 当参数不为空是参数会被解析给上层界面   |
| openNewWindow | 异步方法,打开新页面 | 参考**打开H5打开示例** | 回调接收closeWindow返回的数据 |

模块名: cacheApi(全为同步方法)，永久缓存可以跨进程，受限于CacheManager.init(context:
Application,processMode:CacheProcessMode)

temp:临时缓存和WebFragment同生命周期；
memory:内存缓存和APP同生命周期；
其他：永久缓存

| 方法名 | 说明     | 参数                      | 备注                   |
| --- |--------|-------------------------|----------------------|
| saveTempCache | temp缓存 | ```{key:"",value:""}``` |  |
| getTempCache | temp缓存 | key          |  |
| removeTempCache | temp缓存 | key        |  |
| saveMemoryCache | memory缓存 | ```{key:"",value:""}``` |  |
| getMemoryCache | memory缓存 | key          |  |
| removeMemoryCache | memory缓存 | key          |  |
| saveCache | 永久缓存 | ```{key:"",value:""}``` |  |
| getCache | 永久缓存 | key                     |  |
| removeCache | 永久缓存 | key                     |  |





