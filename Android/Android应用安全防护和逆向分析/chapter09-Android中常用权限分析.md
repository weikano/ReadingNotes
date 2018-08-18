### 一、辅助功能权限
```xml
<application>
    <uses-permission android:name="android.permission.BIND_ACCESSIBILITY_SERVICE"/>
    <service android:name=".MyAccessiblityService"
        android:label="@string/accessibility_label">
        <intent-filter>
            <action android:name="android.accessibilityservice.AccessibilityService"/>
        </intent-filter>
    </service>
</application>
```
**风险：可以监听设备当前的窗口变化，分析当前应用的view结构后模拟点击。**
- 模拟社交软件的登录窗口，盗取用户账号密码
- 当监听到当前应用是社交APP时，可以获取当前的聊天记录
- 分析设备中应用的情况，模拟点击任何一个应用进行操作来盗取信息

### 二、设备管理权限
可以快速定位以及擦除设备数据，防止应用被卸载
```xml
<receiver android:name=".AdminReceiver"
    android:label="@string/blahblah"
    android:description="@string/blahblah"
    android:permission="android.permission.BIND_DEVICE_ADMIN">
    <meta-data android:name="android.app.device_admin"
        android:resource="@xml/device_admin"/>
    <intent-filter>
        <action android:name="android.app.action.DEVICE_ADMIN_ENABLED"/>
    </intent-filter>    
</receiver>
```
**风险：可以修改设备的锁屏密码、擦除数据等**

### 三、通知栏管理权限
辅助功能同样可以监听设备的通知栏情况，通知栏管理权限是专门用来监听设备的通知栏信息。
```xml
<service android:name=".NotificationListener"
    android:label="@string/blahblah"
    android:permission="android.permission.BIND_NOTIFICATION_LISTENER_SERVICE">
    <intent-filter>
        <action android:name="android.service.notification.NotificationListenerService"/>
    </intent-filter>
</service>
```
**风险：设备的通知栏信息会被它接管，造成敏感信息泄露**

### 四、VPN开发权限
使用VpnService开发vpn相关的功能
```xml
<service android:name=".MyVPNService"
    android:permission="android.permission.BIND_VPN_SERVICE">
    <intent-filter>
        <action android:name="android.net.VpnService"/>
    </intent-filter>
</service>
```
**风险：设备的网络权访问消息被接管**
