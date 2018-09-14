# YouTube Android Play API Research

##Reference
>*一盏灯， 一片昏黄； 一简书， 一杯淡茶。 守着那一份淡定， 品读属于自己的寂寞。 保持淡定， 才能欣赏到最美丽的风景！ 保持淡定， 人生从此不再寂寞。*

## Download Api：
> *[地址](https://developers.google.com/youtube/android/player/downloads/)*

##Register your App：
1. ##### Need：
    >***1：需要apk文件在Google API控制台中注册应用程序，包括数字签名文件的公共证书才能使用YouTube Android Player API.***

    >***2：[Google API控制台地址](https://console.developers.google.com/projectselector/apis/credentials?supportedpurview=project)***

2. ##### Credentials：
    >***1：OAuth 2.0:每当应用程序请求私人用户数据时，它都必须随请求一起发送OAuth 2.0令牌。应用程序首先发送一个客户端ID，可能还有一个客户端秘钥来获取令牌。可以为Web应用程序，服务账号或者已安装的应用程序生成OAuth 2.0凭证。***

    >***2：API秘钥：在未提供OAuth 2.0令牌的请求必须发送API秘钥。该秘钥表示项目提供API访问权限。***
    
    >*该API支持对API秘钥的几种类型。如果所需要的API秘钥尚不存在，通过点击创建证书>API秘钥的在控制台中创建一个API秘钥。*
    
    *ideal：注册应用程序时，请确保项目添加了YouTube Data API v3服务，YouTube Android 播放器API依赖数据API来检测YouTube内容。*
    
    ##Setup instructions
    >*创建新项目时，需要将客户端库YouTubeAndroidAPIPlayApi.jar文件导入<projiect_root>/libs目录中，以便将该库包含在构建路径中。也可以手动将.jar文件添加到自己的构建路径中。*
    
    >***1：打开包中的DeveloperKey.java文件YouTubeAndroidAPIDemo,将null改为YouTube开发人员秘钥替换： ***
    
        public static final String DEVELOPER_KEY = null;
        
    >***如果没有设置开发秘钥，那么会抛出java.lang.NullPointerException。***
    
    
    
## Sample Applications  

    1.Video wall
    
    2.Simple PlayView
    
    3.Simple PlayFragement 
    
    4.Custom Player Controls
    
    5.Custom Fullscreen Handling
    
    6.Overlay ActionBar Demo
    
    7.Standalone Player
    
    8.YouTube App Launcher Intents
       
    




















