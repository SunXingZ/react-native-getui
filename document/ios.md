# iOS 手动集成方式

在 react-native link 之后，打开 iOS 工程。
Xcode 工程中需要注册个推 SDK 、注册 deviceToken 、监听消息回调，才能正常使用推送服务，只需要通过以下几步即可集成：

1、AppDelegate.h 中添加如下代码，导入头文件并实现两个 Delegate：

````
// 导入头文件
#import <RCTGetuiModule/RCTGetuiModule.h>
#if __IPHONE_OS_VERSION_MAX_ALLOWED >= __IPHONE_10_0
#import <UserNotifications/UserNotifications.h>
#endif
// 以下三个参数需要到个推官网注册应用获得
#define kGtAppId @"iMahVVxurw6BNr7XSn9EF2"
#define kGtAppKey @"yIPfqwq6OMAPp6dkqgLpG5"
#define kGtAppSecret @"G0aBqAD6t79JfzTB6Z5lo5"

// 这里需要实现 UNUserNotificationCenterDelegate,GeTuiSdkDelegate 两个 Delegate
@interface AppDelegate : UIResponder <UIApplicationDelegate,UNUserNotificationCenterDelegate,GeTuiSdkDelegate>

@property (nonatomic, strong) UIWindow *window;

@end
````
2、AppDelegate.m 的didFinishLaunchingWithOptions 方法里面添加如下代码：

````
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions
{
 // 接入个推
	[GeTuiSdk startSdkWithAppId:kGtAppId appKey:kGtAppKey appSecret:kGtAppSecret delegate:self];
 // 注册远程通知
  [self registerRemoteNotification];
}

/** 注册远程通知 */
- (void)registerRemoteNotification {
    /*
     警告：Xcode8的需要手动开启“TARGETS -> Capabilities -> Push Notifications”
     */

    /*
        警告：该方法需要开发者自定义，以下代码根据APP支持的iOS系统不同，代码可以对应修改。
        以下为演示代码，注意根据实际需要修改，注意测试支持的iOS系统都能获取到DeviceToken
     */
    if ([[UIDevice currentDevice].systemVersion floatValue] >= 10.0) {
#if __IPHONE_OS_VERSION_MAX_ALLOWED >= __IPHONE_10_0 // Xcode 8编译会调用
        UNUserNotificationCenter *center = [UNUserNotificationCenter currentNotificationCenter];
        center.delegate = self;
        [center requestAuthorizationWithOptions:(UNAuthorizationOptionBadge | UNAuthorizationOptionSound | UNAuthorizationOptionAlert | UNAuthorizationOptionCarPlay) completionHandler:^(BOOL granted, NSError *_Nullable error) {
            if (!error) {
                NSLog(@"request authorization succeeded!");
            }
        }];

        [[UIApplication sharedApplication] registerForRemoteNotifications];
#else // Xcode 7编译会调用
        UIUserNotificationType types = (UIUserNotificationTypeAlert | UIUserNotificationTypeSound | UIUserNotificationTypeBadge);
        UIUserNotificationSettings *settings = [UIUserNotificationSettings settingsForTypes:types categories:nil];
        [[UIApplication sharedApplication] registerUserNotificationSettings:settings];
        [[UIApplication sharedApplication] registerForRemoteNotifications];
#endif
    } else if ([[[UIDevice currentDevice] systemVersion] floatValue] >= 8.0) {
        UIUserNotificationType types = (UIUserNotificationTypeAlert | UIUserNotificationTypeSound | UIUserNotificationTypeBadge);
        UIUserNotificationSettings *settings = [UIUserNotificationSettings settingsForTypes:types categories:nil];
        [[UIApplication sharedApplication] registerUserNotificationSettings:settings];
        [[UIApplication sharedApplication] registerForRemoteNotifications];
    } else {
        UIRemoteNotificationType apn_type = (UIRemoteNotificationType)(UIRemoteNotificationTypeAlert |
                                                                       UIRemoteNotificationTypeSound |
                                                                       UIRemoteNotificationTypeBadge);
        [[UIApplication sharedApplication] registerForRemoteNotificationTypes:apn_type];
    }
}

````

3、在AppDelegate.m 的didRegisterForRemoteNotificationsWithDeviceToken 方法中注册 DeviceToken，如下所示：

````
- (void)application:(UIApplication *)application
didRegisterForRemoteNotificationsWithDeviceToken:(NSData *)deviceToken {
  NSString *token = [[deviceToken description] stringByTrimmingCharactersInSet:[NSCharacterSet characterSetWithCharactersInString:@"<>"]];
    [GeTuiSdk registerDeviceToken:token];
}
````

4、为了在收到推送点击进入应用能够获取该条推送内容需要在 AppDelegate.m didReceiveRemoteNotification 方法里面添加 [[NSNotificationCenter defaultCenter] postNotificationName:kJPFDidReceiveRemoteNotification object:userInfo] 方法，注意：这里需要在两个方法里面加一个是iOS7以前的一个是iOS7即以后的，如果AppDelegate.m 没有这个两个方法则直接复制这两个方法，如下所示：

````
#pragma mark - APP运行中接收到通知(推送)处理 - iOS 10以下版本收到推送

/** APP已经接收到“远程”通知(推送) - 透传推送消息  */
- (void)application:(UIApplication *)application didReceiveRemoteNotification:(NSDictionary *)userInfo fetchCompletionHandler:(void (^)(UIBackgroundFetchResult result))completionHandler {

    // [ GTSdk ]：将收到的APNs信息传给个推统计
    [GeTuiSdk handleRemoteNotification:userInfo];

  	[[NSNotificationCenter defaultCenter]postNotificationName:GT_DID_RECEIVE_REMOTE_NOTIFICATION object:@{@"type":@"GT_APNS",@"userInfo":userInfo}];

    completionHandler(UIBackgroundFetchResultNewData);
}

#pragma mark - iOS 10中收到推送消息

#if __IPHONE_OS_VERSION_MAX_ALLOWED >= __IPHONE_10_0
//  iOS 10: App在前台获取到通知
- (void)userNotificationCenter:(UNUserNotificationCenter *)center willPresentNotification:(UNNotification *)notification withCompletionHandler:(void (^)(UNNotificationPresentationOptions))completionHandler {
  	[[NSNotificationCenter defaultCenter]postNotificationName:GT_DID_RECEIVE_REMOTE_NOTIFICATION object:@{@"type":@"GT_APNS",@"userInfo":notification.request.content.userInfo}];
    completionHandler(UNNotificationPresentationOptionBadge | UNNotificationPresentationOptionSound | UNNotificationPresentationOptionAlert);
}

//  iOS 10: 点击通知进入App时触发
- (void)userNotificationCenter:(UNUserNotificationCenter *)center didReceiveNotificationResponse:(UNNotificationResponse *)response withCompletionHandler:(void (^)())completionHandler {
    [GeTuiSdk handleRemoteNotification:response.notification.request.content.userInfo];
  	[[NSNotificationCenter defaultCenter]postNotificationName:GT_DID_CLICK_NOTIFICATION object:response.notification.request.content.userInfo];
    completionHandler();
}
#endif

````

5、在 AppDelegate.m 实现 GetuiDelegate 的代理方法并接受推送消息：

````
#pragma mark - GeTuiSdkDelegate

/** SDK启动成功返回cid */
- (void)GeTuiSdkDidRegisterClient:(NSString *)clientId {
    [[NSNotificationCenter defaultCenter]postNotificationName:GT_DID_REGISTE_CLIENTID object:clientId];
}

/** SDK遇到错误回调 */
- (void)GeTuiSdkDidOccurError:(NSError *)error {}


/** SDK收到透传消息回调 */
- (void)GeTuiSdkDidReceivePayloadData:(NSData *)payloadData andTaskId:(NSString *)taskId andMsgId:(NSString *)msgId andOffLine:(BOOL)offLine fromGtAppId:(NSString *)appId {
    [GeTuiSdk sendFeedbackMessage:90001 andTaskId:taskId andMsgId:msgId];
    // 数据转换
    NSString *payloadMsg = nil;
    if (payloadData) {
        payloadMsg = [[NSString alloc] initWithBytes:payloadData.bytes length:payloadData.length encoding:NSUTF8StringEncoding];
    }
    NSString *msg = [NSString stringWithFormat:@"taskId=%@,messageId:%@,payloadMsg:%@%@", taskId, msgId, payloadMsg, offLine ? @"<离线消息>" : @""];
    NSDictionary *userInfo = @{@"taskId":taskId,@"msgId":msgId,@"payloadMsg":payloadMsg,@"offLine":offLine?@"YES":@"NO"};
    [[NSNotificationCenter defaultCenter]postNotificationName:GT_DID_RECEIVE_REMOTE_NOTIFICATION object:@{@"type":@"GT_PAYLOAD",@"userInfo":userInfo}];
}

/** SDK收到sendMessage消息回调 */
- (void)GeTuiSdkDidSendMessage:(NSString *)messageId result:(int)result {}

/** SDK运行状态通知 */
- (void)GeTuiSDkDidNotifySdkState:(SdkStatus)aStatus {}

/** SDK设置推送模式回调 */
- (void)GeTuiSdkDidSetPushMode:(BOOL)isModeOff error:(NSError *)error {}

````

#JS 使用及接口

主要的消息通知回调使用如下，其他的接口均可在 [index.js](https://github.com/SunXingZ/react-native-getui/blob/master/index.js) 查看。

````
//订阅消息通知
   var { NativeAppEventEmitter } = require('react-native');
   var GT_REGISTER_CLIENT_ID = NativeAppEventEmitter.addListener(
         'GT_REGISTER_CLIENT_ID',
         (clientId) => {
           Alert.alert('GT_REGISTER_CLIENT_ID', clientId);
         }
       )
   var GT_RECEIVE_REMOTE_NOTIFICATION = NativeAppEventEmitter.addListener(
      'GT_RECEIVE_REMOTE_NOTIFICATION',
      (notification) => {
        //消息类型分为 GT_APNS 和 GT_PAYLOAD 透传消息，具体的消息体格式会有差异
        switch (notification.type) {
            case "GT_APNS":
                Alert.alert('GT_APNS 消息通知',JSON.stringify(notification))
                break;
            case "GT_PAYLOAD":
                Alert.alert('GT_PAYLOAD 消息通知',JSON.stringify(notification))
                break;
            default:
        }
      }
    );

    var GT_CLICK_REMOTE_NOTIFICATION = NativeAppEventEmitter.addListener(
        'GT_CLICK_REMOTE_NOTIFICATION',
        (notification) => {
            Alert.alert('GT_CLICK_REMOTE_NOTIFICATION',JSON.stringify(notification))
        }
    );

    <!-- VoIP 推送通知回调 -->

    var GT_VOIP_PUSH_PAYLOAD = 
        NativeAppEventEmitter.addListener(
            'GT_VOIP_PUSH_PAYLOAD',
            (notification) => {
                Alert.alert('GT_VOIP_PUSH_PAYLOAD ',JSON.stringify(notification))
            }
        );
````
**注意** 

为保证正确收到 VoIP 推送回调，需要先调用注册 VoIP 接口 `Getui.voipRegistration()`，并且需要打开推送统治权限，并且开启 VoIP 后台运行权限。

![VoIP 权限配置](https://github.com/SunXingZ/react-native-getui/blob/master/document/img/ios_1.jpeg?raw=true)

````
componentWillUnMount() {
  //记得在此处移除监听
    GT_RECEIVE_REMOTE_NOTIFICATION.remove()
    GT_CLICK_REMOTE_NOTIFICATION.remove()
    GT_REGISTER_CLIENT_ID.remove()
    GT_VOIP_PUSH_PAYLOAD.remove()
}
````

其他接口：

````
import Getui from 'react-native-getui'

  // 注册 VoIP 通知，只有注册后才能收到 VoIP 通知。
  Getui.voipRegistration();
  
  Getui.clientId((param)=> {
       this.setState({'clientId': param})
   })

   Getui.version((param)=> {
       this.setState({'version': param})
   })

   Getui.status((param)=> {
       let status = ''
       switch (param){
           case '0':
               status = '正在启动'
               break;
           case '1':
               status = '启动'
               break;
           case '2':
               status = '停止'
               break;
       }
       this.setState({'status': status})
   })
//Getui...

````
