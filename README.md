前段时间在项目中遇到了一个问题：从原生模块跳转到RN模块时会有一段短暂的白屏时间，特别是在低端手机更加明显。在网上搜了一圈，发现这个问题非常常见。

``` java
ReactRootView mReactRootView = createRootView();
mReactRootView.startReactApplication(mReactInstanceManager, getMainComponentName(), getLaunchOptions());
```
这两行代码就是白屏的主要原因。因为这两行代码把jsbundle文件读入到内存中，这个过程肯定是需要耗费一些时间的，当jsbundle文件越大，可以预见加载到内存中需要的时间就越长。
解决办法就是**以空间换时间**，在app启动时候，就将ReactRootView初始化出来，并缓存起来，在用的时候从缓存获取ReactRootView使用，达到秒开。
目前的React Native版本更新到了0.45.1，而网上大部分的解决方案都偏旧,但是解决思路还是一样的，不过具体的解决方法会做些修改（因为RN源码的变动）。
下面会详细说明。
##  1. 创建ReactRootView缓存管理器

``` java
public class RNCacheViewManager {
    public static final int REQUEST_OVERLAY_PERMISSION_CODE = 1111;
    public static final String REDBOX_PERMISSION_MESSAGE =
            "Overlay permissions needs to be granted in order for react native apps to run in dev mode";
    private static ReactRootView mRootView = null;

    public static ReactRootView getRootView() {
        if (mRootView == null) {
            throw new RuntimeException("缓存view管理器尚未初始化！");
        }
        return mRootView;
    }


    public static ReactNativeHost getReactNativeHost(Activity activity) {
        return ((ReactApplication) activity.getApplication()).getReactNativeHost();
    }

    public static void init(Activity act, String moduleName, Bundle lauchOptions) {
        boolean needsOverlayPermission = false;
        if (mRootView != null) {
            return;
        }
        if (BuildConfig.DEBUG && Build.VERSION.SDK_INT >= Build.VERSION_CODES.M && !Settings.canDrawOverlays(act)) {
            needsOverlayPermission = true;
            Intent serviceIntent = new Intent(Settings.ACTION_MANAGE_OVERLAY_PERMISSION, Uri.parse("package:" + act.getPackageName()));
            FLog.w(ReactConstants.TAG, REDBOX_PERMISSION_MESSAGE);
            Toast.makeText(act, REDBOX_PERMISSION_MESSAGE, Toast.LENGTH_LONG).show();
            act.startActivityForResult(serviceIntent, REQUEST_OVERLAY_PERMISSION_CODE);
        }
        if (!needsOverlayPermission) {
            mRootView = new ReactRootView(act);
            mRootView.startReactApplication(
                    getReactNativeHost(act).getReactInstanceManager(),
                    moduleName,
                    lauchOptions);
        }
    }

    /**
     * 不能再调用原有的销毁方法，否则rn初始化出来的对象会被销毁,同时
     * 在ReactActivity销毁后，我们需要把 view从父视图中移除。
     */
    public static void onDestroy() {
        try {
            ViewParent parent = getRootView().getParent();
            if (parent != null)
                ((android.view.ViewGroup) parent).removeView(getRootView());
        } catch (Throwable e) {
            e.printStackTrace();
        }
    }

}
```
##  2. 继承ReactActivity和ReactActivityDelegate做相应修改
第二步就是与之前不太一样的地方，因为现在ReactActivity的主要逻辑基本都由ReactActivityDelegate代理实现,所以所做的修改就有所不同，只需要实现自己的代理并在自己的ReactActivity覆盖createReactActivityDelegate即可。


### 继承ReactActivityDelegate
这里直接继承ReactActivityDelegate并复写**onCreate方法**，**loadApp**方法，以及**onDestroy**方法。

``` java
public class CCCReactActivityDelegate extends ReactActivityDelegate {
    private Activity mActivity;

    public CCCReactActivityDelegate(Activity activity, @Nullable String mainComponentName) {
        super(activity, mainComponentName);
        mActivity = activity;
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        Class<ReactActivityDelegate> clazz=ReactActivityDelegate.class;
        try {
            Field field=clazz.getDeclaredField("mDoubleTapReloadRecognizer");
            field.setAccessible(true);
            field.set(this, new DoubleTapReloadRecognizer());
        } catch (Exception e) {
            e.printStackTrace();
        }
        if (BuildConfig.DEBUG && Build.VERSION.SDK_INT >= Build.VERSION_CODES.M&&!Settings.canDrawOverlays(mActivity)) {
            TextView textView=new TextView(mActivity);
            textView.setLayoutParams(new LinearLayout.LayoutParams(LinearLayout.LayoutParams.WRAP_CONTENT,LinearLayout.LayoutParams.WRAP_CONTENT));
            textView.setText(RNCacheViewManager.REDBOX_PERMISSION_MESSAGE+"\nPlease exit the app and grant again");
            textView.setTextColor(Color.rgb(255, 0, 0));
            mActivity.setContentView(textView);
        }else{
            loadApp(null);
        }
    }

    @Override
    protected void onDestroy() {
        RNCacheViewManager.onDestroy();
        super.onDestroy();
    }

    @Override
    protected void loadApp(String appKey) {
        ReactRootView mReactRootView = RNCacheViewManager.getRootView();
        ViewParent viewParent = mReactRootView.getParent();
        if (viewParent != null) {
            ViewGroup vp = (ViewGroup) viewParent;
            vp.removeView(mReactRootView);
        }
        mActivity.setContentView(mReactRootView);
    }
}
```
重点关注`onCreate`方法与`loadApp`以及`onDestroy`方法。`onCreate`方法中调用了loadApp方法，因为此时不需要appkey(appkey在缓存管理器初始化的时候就使用了)所以直接传空即可.`onDestroy`使用RNCacheViewManager.onDestroy()来移除ReactRootView。

### 继承ReactActivity并覆盖代理创建方法

``` java
public  class CCCReactActivity extends ReactActivity  {
    @Override
    protected ReactActivityDelegate createReactActivityDelegate() {
        return new CCCReactActivityDelegate(this,null);
    }
}
```
重写ReactActivity的地方很少，即是替换掉原来的ReactActivityDelegate。
## 3. 创建React Native对应的Activity
在这里可以像之前继承ReactActivit那样创建自己的Activity(继承CCCReactActivity)，覆盖自己需要的方法，也可以直接使用CCCReactActivity。

## 4. 在App第一个Activity启动时候初始化ReactRootView缓存管理器

``` kotlin
RNCacheViewManager.init(this, "这里填写模块名", null);
```
需要注意的是：
第三个参数可以设置传递给RN的属性（Bundle封装类型），如有需要才传值，否则传空即可。
如果应用只有一个Activity，加载RN还是比较难避免白屏(因为可能会打开这个Activity立即跳转RN页面，此时JsBundle还未完全加载到内存)。因此最好有个启动界面。

## 5.对比测试
在三星SM-G3609手机(运存768M)上做了几次测试，打包后的jsBundle大小：522KB
无预加载的情况下，从原生模块打开RN页面平均耗时1769 ms
有预加载的情况下，从原生模块打开RN页面平均耗时160ms
效果非常明显！
从用户体验来说，打开页面如果有1 2秒白屏这简直不能忍，而通过预加载可以达到几乎是秒开的体验，所以为什么不用呢？

## 6.关于targetSdkVersion 23以上的SYSTEM_ALERT_WINDOW权限问题(7.5更新)
如果工程是在调试模式并且targetSdkVersion在23以上，打开应用就会报如下错误：

``` stylus
 android.view.WindowManager$BadTokenException: Unable to add window android.view.ViewRootImpl$W@dac42 -- permission denied for this window type
```
在调试模式并且targetSdkVersion在23以上的工程中，如果未授权就去执行startReactApplication就会报BadTokenException错误。因此我们需要对启动React Native加一些必要的条件判断。

``` java
        boolean needsOverlayPermission = false;
        if (mRootView != null) {
            return;
        }
        if (BuildConfig.DEBUG && Build.VERSION.SDK_INT >= Build.VERSION_CODES.M && !Settings.canDrawOverlays(act)) {
            needsOverlayPermission = true;
            Intent serviceIntent = new Intent(Settings.ACTION_MANAGE_OVERLAY_PERMISSION, Uri.parse("package:" + act.getPackageName()));
            FLog.w(ReactConstants.TAG, REDBOX_PERMISSION_MESSAGE);
            Toast.makeText(act, REDBOX_PERMISSION_MESSAGE, Toast.LENGTH_LONG).show();
            act.startActivityForResult(serviceIntent, REQUEST_OVERLAY_PERMISSION_CODE);
        }
        if (!needsOverlayPermission) {
            mRootView = new ReactRootView(act);
            mRootView.startReactApplication(
                    getReactNativeHost(act).getReactInstanceManager(),
                    moduleName,
                    lauchOptions);
        }
```
官方做法是打开React Native承载的Activity才去申请权限以及接收权限是否授予都在同一个Activity中处理，而预加载方法则是在应用可能会在启动时就开始申请权限，因此也建议在申请所在的Activity接收权限是否授予回调，否则即使授予了权限也不会去执行`startReactApplication`方法了，此时只能杀死应用重新打开才能加载RN了。
如果条件允许则在申请所在的Activity接收权限是否授予回调，即覆盖`onActivityResult`方法，如果被授权则会开始加载React Native，具体代码如下：

``` java
    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        super.onActivityResult(requestCode, resultCode, data);
        if (requestCode == REQUEST_OVERLAY_PERMISSION_CODE && Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            if (Settings.canDrawOverlays(this)) {
                RNCacheViewManager.getRootView().startReactApplication(
                        RNCacheViewManager.getReactNativeHost(this).getReactInstanceManager(),
                        MODULE_NAME,
                        null);
            }
        }
    }
```
而如果是在SplashActivtiy中加载RN，因为SplashActivtiy会自动关闭，等到授权完成了可能Activity已经不在了，此时
onActivityResult也不会起作用了，这时候只能授权并杀死应用重新启动了。

## 7.局限性
如果在init时没有需要传递的属性(lauchOptions)，`init`方法当然是早一点调用比较好，例如App启动时就调用该方法，
如果需要传递属性(lauchOptions)，而lauchOptions需要比较迟才能获取到，就只能在RN页面打开加载之前去init了(强烈不建议这种做法，预加载几乎无作用)。因此建议lauchOptions最好只传递尽可能早明确的属性，例如一些appkey配置等。而如果需要通过传递属性来动态选择RN加载的页面，这种多入口的方式就不合适选择预加载，此时更推荐选择多注册的方式来实现多入口。
此外，以上实现方式只针对Activity中加载RN，如果需要在Fragment中加载则需要选择另外一种实现，下回分解。



项目源码：[RNPreloadBundle][1]

运行方法：

 1. 在项目根目录下使用`npm i`安装node_modules依赖
 2. 在项目根目录下使用react-native start 命令启动react-native服务器
 3. 运行项目安装应用

参考文章：
[ReactNative安卓首屏白屏优化][2]


  [1]: https://github.com/LittleLiByte/RNPreloadBundle.git
  [2]: https://github.com/cnsnake11/blog/blob/master/ReactNative%E5%BC%80%E5%8F%91%E6%8C%87%E5%AF%BC/ReactNative%E5%AE%89%E5%8D%93%E9%A6%96%E5%B1%8F%E7%99%BD%E5%B1%8F%E4%BC%98%E5%8C%96.md
