# Android 手动集成方式

在 react-native link 之后， 使用Android Studio打开工程（假设主模块名称为app），当然您也可以不适用Android Studio。

1、如果您使用Android Studio作为开发Android原生代码的IDE，使用 Android Studio import 你的 React Native 应用（选择你的 React Native 应用所在目录下的 android 文件夹即可）

2、android/app/build.gradle 的defaultConfig节点中添加如下代码

````
// 配置个推的三个参数
manifestPlaceholders = [
            GETUI_APP_ID : "你的AppID",
            GETUI_APP_KEY : "你的AppKey",
            GETUI_APP_SECRET : "你的AppSecret"
        ]

````

3、在android/app/AndroidManifest.xml 的`<application>`标签内添加meta-data

````
<!-- 配置个推的三个参数 -->
<meta-data android:name="PUSH_APPID" android:value="${GETUI_APP_ID}" />
<meta-data android:name="PUSH_APPKEY" android:value="${GETUI_APP_KEY}" />
<meta-data android:name="PUSH_APPSECRET" android:value="${GETUI_APP_SECRET}" />

````

4、在android/app/build.gradle的`dependencies`节点中添加对react-native-getui的依赖（如果已经存在，忽略此步）

````
compile project(':react-native-getui')

````

5、在android/app/src/main/{您的包名}/MainActivity.onCreate中初始化个推初始化代码

````
GetuiModule.initPush(this);

````
如果您的MainActivity方法中未重写onCreate方法，请先重写onCreate方法。

````
protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        GetuiModule.initPush(this);
    }
````
如果您使用Android Studio作为IDE，Android Studio会自动为您import 相应的类名，如果您使用其他的IDE，请import相关的类

````
import android.os.Bundle;
import com.getui.reactnativegetui.GetuiModule;
````

#JS 使用

主要的消息通知回调使用如下，其他的接口均可在 [index.js](https://github.com/SunXingZ/react-native-getui/blob/master/index.js) 查看。

````
//订阅消息通知
   var { NativeAppEventEmitter } = require('react-native');
   var GT_RECEIVE_REMOTE_NOTIFICATION = NativeAppEventEmitter.addListener(
      'GT_RECEIVE_REMOTE_NOTIFICATION',
      (notification) => {
        //消息类型分为 cmd 和 payload 透传消息，具体的消息体格式会有差异
        switch (notification.type) {
            case "GT_CLIENT_ID":
                Alert.alert('GT_CLIENT_ID',JSON.stringify(notification))
                break;
            case "GT_CMD":
                Alert.alert('GT_CMD消息通知',JSON.stringify(notification))
                break;
            case "GT_PAYLOAD":
                Alert.alert('GT_PAYLOAD消息通知',JSON.stringify(notification))
                break;
            //新增回调通知到达，点击回调
            case 'GT_NOTIFICATION_ARRIVED':
                Alert.alert('GT_NOTIFICATION_ARRIVED通知到达',JSON.stringify(notification))
                break
            case 'GT_NOTIFICATION_CLICKED':
                Alert.alert('GT_NOTIFICATION_CLICKED通知点击',JSON.stringify(notification))
                break
            default:
        }
      }
    );


````

````
componentWillUnMount() {
  //记得在此处移除监听
    GT_RECEIVE_REMOTE_NOTIFICATION.remove()
}

````

其他接口：

````
import Getui from 'react-native-getui'

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
