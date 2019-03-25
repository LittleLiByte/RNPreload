
# How to use
react-native-preload-library的集成是在原生Android项目集成React Native基础上添加必要的配置来完成。
## 1.在原生项目根目录下添加package.json和index.android.js文件

package.json文件基本配置如下

``` json
{
  "name": "PreLoadRN",
  "version": "0.0.1",
  "private": true,
  "scripts": {
    "start": "node node_modules/react-native/local-cli/cli.js start",
    "test": "jest"
  },
  "dependencies": {
    "react": "16.0.0-alpha.6",
    "react-native": "0.44.3",
    "react-native-preload-library": "^1.0.0"
  },
  "devDependencies": {
    "jest": "20.0.4",
    "react-test-renderer": "16.0.0-alpha.6"
  },
  "jest": {
    "preset": "react-native",
    "setupFiles": [
      "./jest/setup.js"
    ]
  }
}
```
其中react和react-native可以根据最新版走，  `"react-native-preload-library": "^1.0.0"`则是必要的预加载库依赖的配置。
index.android.js文件根据实际情况填写。
## 2.安装npm依赖包
使用以下命令`npm i` 来安装所以依赖
或者在原有基础上通过`npm i --save react-native-preload-library`单独安装react-native-preload-library依赖包。

## 3.修改原生项目，引入 react-native-preload-library模块

``` stylus
include ':react-native-preload-library'
project(':react-native-preload-library').projectDir = new File(rootProject.projectDir, './node_modules/react-native-preload-library/android')
```

## 4.修改Project的build.gradle如下：

``` dts
allprojects {
    repositories {
        jcenter()
        maven {
            // All of React Native (JS, Android binaries) is installed from npm
            url "$rootDir/node_modules/react-native/android"
        }
    }
}
```
这是更改依赖React Native引入路径，直接从node_modules中获取，而不从maven仓库中下载，maven仓库中的React Native已经很久没更新了。

## 5.修改app的build.gradle

``` groovy
compile 'com.facebook.react:react-native:0.44.3'
debugCompile project(path: ':react-native-preload-library', configuration: 'allDebug')
releaseCompile project(path: ':react-native-preload-library', configuration: 'allRelease')
```
请注意，react-native:0.44.3是在package.json中指定的版本，如果用的是其他版本，这里的版本号也需要做相应的修改。
下面两行代码则是因为需要在library模块中区分当前是debug模式还是release模式而配置。


## 6.创建MainApplication

``` java
public class MainApplication extends Application implements ReactApplication {


    public final ReactNativeHost mReactNativeHost = new ReactNativeHost(this) {

        @Override
        public boolean getUseDeveloperSupport() {
            return BuildConfig.DEBUG;
        }

        @Override
        protected List<ReactPackage> getPackages() {
            return Arrays.<ReactPackage>asList(
                    new MainReactPackage()
            );
        }
    };

    @Override
    public ReactNativeHost getReactNativeHost() {
        return mReactNativeHost;
    }
}
```


## 7.创建CCCReactActivity或CCCReactFragment子类

 - 如果需要创建一个加载RN的Acitvity，则继承CCCReactActivity并重写`getMainComponentName`方法，注意此处的MODULE_NAME属性是为了方便引用才这么写。
``` java
public class RNExampleActivity extends CCCReactActivity {
    public static final String MODULE_NAME = "PreLoadRNActivity";

    @Override
    public String getMainComponentName() {
        return MODULE_NAME;
    }
}
```

 - 如果需要创建一个加载RN的Fragment，则继承CCCReactFragment并重写`getMainComponentName`方法，注意此处的MODULE_NAME属性是为了方便引用才这么写。

``` scala
 public class ExampleFragment extends CCCReactFragment {
    public static final String MODULE_NAME = "PreLoadRNFragment";

    @Override
    public String getMainComponentName() {
        return MODULE_NAME;
    }
}
```
此外，装载这个特殊的Fragment同样需要特殊的Activity--CCCReactFragmentActivity，例子如下：

``` java
public class RNExampleFragmentActivity extends CCCReactFragmentActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_container);
        Fragment exampleFragment = new ExampleFragment();
        getFragmentManager().beginTransaction().add(R.id.container, exampleFragment).commit();
    }
}
```

## 8.在app启动的第一个Activity初始化预加载RN
示例如下：

``` java
public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        RNCacheViewManager.init(this, null, RNExampleActivity.MODULE_NAME, ExampleFragment.MODULE_NAME);
    }

    public void gotoRNActivity(View view) {
        startActivity(new Intent(this, RNExampleActivity.class));
    }

    public void gotoRNFragment(View view) {
        startActivity(new Intent(this, RNExampleFragmentActivity.class));
    }

    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {
        super.onActivityResult(requestCode, resultCode, data);
        if (requestCode == REQUEST_OVERLAY_PERMISSION_CODE && Build.VERSION.SDK_INT >= Build.VERSION_CODES.M && Settings.canDrawOverlays(this))
            for (Map.Entry<String, ReactRootView> entry : CACHE.entrySet())
                RNCacheViewManager.getRootView(entry.getKey()).startReactApplication(
                        RNCacheViewManager.getReactNativeHost(this).getReactInstanceManager(),
                        entry.getKey(),
                        null);
    }
}
```
 重点在于：`RNCacheViewManager.init(this, null, RNExampleActivity.MODULE_NAME, ExampleFragment.MODULE_NAME);`这一句，这里初始化需要预加载的RN模块。
 
重写`onActivityResult`因为RN在**Debug模式**下需要用到系统弹窗这个权限，因此在初始化之前就会申请授权弹出个授权页面，如果授权成功则需要在此处重新初始化。而如果在授权成功之前这个页面就被关闭了，则需要杀死应用重新启动了。而在Release模式下则不需要授权。

## 9.添加权限
``` xml
  <uses-permission android:name="android.permission.INTERNET"/>
    <uses-permission android:name="android.permission.SYSTEM_ALERT_WINDOW"/>
```
## 10.启动RN服务，运行
