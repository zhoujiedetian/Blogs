1. Universal Link 是什么？
    

Universal Link 就是让一个 HTTPS 链接既能打开网页，也能在已安装 App 时直达 App 内页面。

2. Universal Link 配置流程
    

- 打开 AppId 的 Associated Domains 功能
    

![[C60A0554-E1C7-46F7-9BF4-B6536FD13C42.png]]

- 在 Xcode 工程中，点击 target -> Signing & Capabilities -> +Capability 添加 Associated Domains
    

![[88ABD486-EC5E-425D-8D2F-4DFC01708CFE.png]]

- 在Associated Domains 中添加对应的 univeral link 地址，格式为applinks: [+](http://security.cloud.tenda.com.cn/) 域名
    
- 生成一个 apple-app-site-association 文件，不要带任何后缀名，内容为
    

{

"applinks": {

"apps": [],

"details": [

{

"appID": "TeamId.AppBundleId",

"paths": ["*"]

}

]

}

}

- appID = TeamId + BundleID
    
- paths 为域名+path 可以打开你的 app，只配一个通配符即代表整个域名都能打开你的 app
    

- 让运维小伙伴将apple-app-site-association 文件放到域名根目录下
    

3. Universal Link 相关验证
    

- 在备忘录中输入域名+apple-app-site-association，然后长按这个链接，能够预览这个文件
    

![[452E9DAB-9A50-4DDB-ABEE-06F550DDEF0E.png]]

- 在备忘录中输入域名（通用链接），然后长按这个链接，弹出的选项有在 App 内打开
    

![[B96E6340-8801-443C-99BB-71C8CFE97239.png]]

4. 微信开放平台配置 Universal Link
    

登录微信开放平台 -> 管理中心 -> 选择对应的移动应用 -> 配置 Universal Link 链接

![[161E1D23-08C6-4E81-BFEA-FF8AB4E60EE1.png]]

5. **注意事项**
    

- App 只会在第一次启动的时候会去拉 AASA 文件，所以测试要删除 App 再重新安装
    
- App 拉 AASA 文件是在走 [https://app-site-association.cdn-apple.com](https://app-site-association.cdn-apple.com) 这个链接去拉的，要保证这个链接能正确调用，若手机网络被 Charles 等抓包软件代理了，会造成文件拉取失败
    

可能出现的坑

Q: 调用 分享、登录、打开小程序 api 唤起微信后又自动返回到 APP

A: 开发者在 registerApp 传入的 Universal links 不生效。

Universal links 失效的可能原因：

1、工程配置 Associated Domain 未打开或未添加 Universal links 域名

2、未在 微信开放平台 配置 Universal links 域名

3、apple-app-site-association 未上线或未按苹果要求放在服务器指定的路径下(域名根目录)

4、apple-app-site-association 的 Universal links 的 path 末尾没有加通配符*

5、apple-app-site-association 的 appID（teamID+bundleID） 与实际不符

6、apple-app-site-association 所在站点必须支持https且不支持重定向

7、apple-app-site-association 只在 app 第一次启动时才会去下载 apple-app-site-association 文件，所以请删除 app 重新安装

8、工程配置 Associated Domain 的格式不正确，必须以 applinks: 开头

9、未重写 AppDelegate 或 SceneDelegate 的 continueUserActivity 方法。

建议使用自检函数排查，从微信SDK1.8.7版本开始（截止发文微信SDK版本为1.9.2），WXApi 新增了自检函数 checkUniversalLinkReady:，帮助开发者排查 SDK 接入过程中遇到的问题。

[WXApi startLogByLevel:WXLogLevelDetail logBlock:^(NSString * _Nonnull log) {

NSLog(@"zjdev wxapi WXLogLevelDetail log=%@", log);

}];

BOOL result = [WXApi registerApp:WX_APP_ID universalLink:@"https://www.tenda.com.cn/"];

NSLog(@"zjdev wxapi result %@", result?@"success":@"fail");

[WXApi checkUniversalLinkReady:^(WXULCheckStep step, WXCheckULStepResult * _Nonnull result) {

NSLog(@"zjdev wxapi checkUniversalLinkReady step=%ld(%@), success=%d, errorInfo=%@, suggestion=%@",

(long)step,

[self wxULCheckStepName:step],

result.success,

result.errorInfo,

result.suggestion);

}];

补充

path 的命名规则如下

"paths": [ "/wwdc/news/", "NOT /videos/wwdc/2010/*", "/videos/wwdc/201?/*"]

*：通配符

?: 占位符

path 路径大小写敏感

Universal Link 官网 [https://developer.apple.com/library/archive/documentation/General/Conceptual/AppSearch/UniversalLinks.html#//apple_ref/doc/uid/TP40016308-CH12-SW1](https://developer.apple.com/library/archive/documentation/General/Conceptual/AppSearch/UniversalLinks.html#//apple_ref/doc/uid/TP40016308-CH12-SW1)

AASA校验

[https://branch.io/resources/aasa-validator/#resultsbox](https://branch.io/resources/aasa-validator/#resultsbox)