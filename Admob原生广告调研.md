Admob原生广告调研

## 一，调研背景：

​	Admob 原生广告（AppInstallAd） 跳转方式改变。

​		跳转方式：原：跳转到手机Google play 的MianActivity。

​				  现：启动Google play 的InlineAppDetailsDialog。

## 二，调研结果：

​	Admob原生广告的跳转方式改变是由Google play或者Admob SDK导致的。

## 三，Admob介绍：

​	Admob是全球广告客户需求量最大的来源，灵活的广告控制和行业领先的中介服务，是通过应用程序赚钱

并通过应用程序广告最大化您的应用程序收入的最佳平台。

Admob广告分为4种广告形式：（1），Banner 样式广告：横幅广告是矩形图片或者文字广告，占据应用布局

中的一个位置，当用户与应用程序进行交互时，它们将保持在屏幕上，并在一段时间后自动刷新。（2），

Native （原生广告）：是一种广告素材资源通过本地平台的UI组件呈现给用户的格式，它是使用您构建的布局

来显示，在编码方面，这意味着当原生广告加载时，您的应用程序会受到一个NativeAd包含其资产的对象，然

后我们的布局负责显示它们（而不是SDK）。

（3），Interstitial（插屏式广告）：插页式广告涵盖其主机应用的全屏广告。它们通常显示在应用的使用过程

中出现。例如在活动或者游戏中各层之间的暂停期间。当应用显示插页式广告时，用户可以选择点击广告并继

续到达内容或者关闭并且返回到应用。（4），Rewarded Video（奖励式广告）：奖励视频广告是全屏视频广

告，用户可以选择全程观看，以换取应用内奖励。



## 四，接入方法：

​	先决条件：~使用Android Studio 1.0或者更高版本。~目标Android API级别14或者更高。~建议：创建

一个AdMob账号并注册一个应用程序。

​	1，导入移动广告SDK：在项目级别build.gradle。

```
maven {url "https://maven.google.com"}
```

​	2，在应用级别的build.gradle。

```
 compile 'com.google.android.gms:play-services-ads:11.8.0'
```

​	3，初始化MobileAds：在加载广告之前，让您的应用程序通过MobileAds.initialize()使用您的AdMob应用程

序ID进行初始化Mobile Ads SDK。这只需要做一次，理想状态下在应用程序启动。

```
package ...
import ...
import com.google.android.gms.ads.MobileAds;

public class MainActivity extends AppCompatActivity {
    ...
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);

        // Sample AdMob app ID: ca-app-pub-3940256099942544~3347511713
        MobileAds.initialize(this, "YOUR_ADMOB_APP_ID");
    }
    ...
}
```

​	4，通过AdLoader类加载广告，该类有自己的AdLoader.Builder类在创建过程中对其进行自定义。通过

AdLoader在构建的过程中添加侦听器。

```
AdLoader adLoader = new AdLoader.Builder(context, "ca-app-pub-3940256099942544/2247696110")
    .forAppInstallAd(new OnAppInstallAdLoadedListener() {
        @Override
        public void onAppInstallAdLoaded(NativeAppInstallAd appInstallAd) {
            // Show the app install ad.
        }
    })
    .forContentAd(new OnContentAdLoadedListener() {
        @Override
        public void onContentAdLoaded(NativeContentAd contentAd) {
            // Show the content ad.
        }
    })
    .withAdListener(new AdListener() {
        @Override
        public void onAdFailedToLoad(int errorCode) {
            // Handle the failure by logging, altering the UI, and so on.
        }
    })
    .withNativeAdOptions(new NativeAdOptions.Builder()
            // Methods in the NativeAdOptions.Builder class can be
            // used here to specify individual options settings.
            .build())
    .build();
    
注释1：forAppInstallAd()：调用此方法会配置AdLoader要求应用安装广告，当广告加载成功时，监听对象的onAppInstaallAdLoad()方法会被调用。
注释2：forContentAd()：此方法forAppInstallAd()与内容广告的作用相同。广告加载成功后，onContentAdLoader()方法将被调用。
```

​	5，加载广告：完成构建以后使用AdLoader来加载广告。有两种方法可用于此：loadAd()和loadAds()。

```
adLoader.loadAd(new AdRequest.Builder().build());
```

```
adLoader.loadAds(new AdRequest.Builder().build(), 3);
注释：该loadAds（）方法发送多个广告的请求（最多请求5个）。
```

注释：4.5整合。

```
final AdLoader adLoader = new AdLoader.Builder(this, ADMOB_AD_UNIT_ID)
        .forAppInstallAd(new NativeAppInstallAd.OnAppInstallAdLoadedListener() {
    @Override
    public void onAppInstallAdLoaded(NativeAppInstallAd ad) {
        ...
        // some code that displays the app install ad.
        ...
        if (adLoader.isLoading()) {
            // The AdLoader is still loading ads.
            // Expect more adLoaded or onAdFailedToLoad callbacks.
        } else {
            // The AdLoader has finished loading ads.
        }
    }
}).build();

adLoader.loadAds(new AdRequest.Builder().build(), 3);
```

​	6，注册广告对象。

```
adView.setNativeAd(ad);
```

Github地址：

​		*原生广告高级实例应用程序：[Java](https://github.com/googleads/googleads-mobile-android-examples/tree/master/java/admob/NativeAdvancedExample)

​		*原生广告高级ListView实例应用程序：[Java](https://github.com/googleads/googleads-mobile-android-examples/tree/master/java/advanced/NativeListViewExample)



## 五，Admob demo 跳转结果。

​	在拉去成功原生广告（安装应用的广告）之后，点击Install跳转到Google play 里的MainActivity，没有跳转

到Google play的InlineAppDetailsDialog 。由此判断跳转结果的改变不是Admob广告导致的。



## 六，聚合广告导致跳转方式的改变？

​	聚合广告也可以排除，因为聚合广告的逻辑代码没有更新过，第二点就是聚合里面的广告的跳转逻辑还是

Admob广告自己的。



## 七，Google play做了某些操作导致的？

​	网页地址：[StackOverflow](https://stackoverflow.com/questions/42734624/open-google-play-dialog-from-my-app)

​	这里提出的的问题是从YouTube，点击展示的Google 广告的时候，弹出对话框（也就是启动

InlineAppDetailsDialog）。这里给我的解释：play 商店应用的这一功能仅开放于部分APP。

具体的解释为：经过反编译Google play，从中得到com.google.android.finsky.activities.InlineAppDetailsDialog,中

存在一个switch方法用来检查调用的包的应用程序ID和签名。只有授权的应用程序才能显示此对话框。具体代码看下面：

```
switch (string2.hashCode()) {
            case 714499313: {
                if (!string2.equals("com.facebook.katana")) break;
                n2 = 0;
                break;
            }
            case 419128298: {
                if (!string2.equals("com.facebook.wakizashi")) break;
                n2 = 1;
                break;
            }
            case -649684660: {
                if (!string2.equals("flipboard.app")) break;
                n2 = 2;
                break;
            }
            case 1249065348: {
                if (!string2.equals("com.kakao.talk")) break;
                n2 = 3;
                break;
            }
            case 1153658444: {
                if (!string2.equals("com.linkedin.android")) break;
                n2 = 4;
                break;
            }
            case -583737491: {
                if (!string2.equals("com.pinterest")) break;
                n2 = 5;
                break;
            }
            case -928396735: {
                if (!string2.equals("com.test.overlay")) break;
                n2 = 6;
                break;
            }
            case 10619783: {
                if (!string2.equals("com.twitter.android")) break;
                n2 = 7;
                break;
            }
            case 1835489205: {
                if (!string2.equals("ru.yandex.weatherplugin")) break;
                n2 = 8;
                break;
            }
            case 19680841: {
                if (!string2.equals("ru.yandex.yandexnavi")) break;
                n2 = 9;
                break;
            }
            case 19650874: {
                if (!string2.equals("ru.yandex.yandexmaps")) break;
                n2 = 10;
                break;
            }
            case 1663191933: {
                if (!string2.equals("ru.yandex.yandexbus")) break;
                n2 = 11;
                break;
            }
            case 636981927: {
                if (!string2.equals("ru.yandex.metro")) break;
                n2 = 12;
                break;
            }
            case 647779725: {
                if (!string2.equals("ru.yandex.searchplugin")) break;
                n2 = 13;
                break;
            }
            case -143313792: {
                if (!string2.equals("ru.yandex.test.promolib")) break;
                n2 = 14;
                break;
            }
            case -2075712516: {
                if (!string2.equals("com.google.android.youtube")) break;
                n2 = 15;
                break;
            }
            case 1387611572: {
                if (!string2.equals("com.google.android.youtube.tv")) break;
                n2 = 16;
                break;
            }
            case 886484461: {
                if (!string2.equals("com.google.android.apps.youtube.kids")) break;
                n2 = 17;
                break;
            }
            case 1386399663: {
                if (!string2.equals("com.google.android.apps.youtube.gaming")) break;
                n2 = 18;
                break;
            }
            case 1713433253: {
                if (!string2.equals("com.google.android.apps.youtube.music")) break;
                n2 = 19;
                break;
            }
            case 1252744364: {
                if (!string2.equals("com.google.android.apps.youtube.creator")) break;
                n2 = 20;
                break;
            }
            case 304833084: {
                if (!string2.equals("com.google.android.apps.youtube.vr")) break;
                n2 = 21;
                break;
            }
            case 1712832578: {
                if (!string2.equals("com.google.android.apps.youtube.mango")) break;
                n2 = 22;
                break;
            }
```





























































###### 











- ​





























