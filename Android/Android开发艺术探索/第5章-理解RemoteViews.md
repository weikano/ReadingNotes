# RemoteViews的应用
## RemoteViews在通知栏上的应用
> Notification

## RemoteViews在桌面小部件上的应用
> 开发步骤

1. 定义小部件界面.
2. 定义小部件配置信息.
> 在res/xml下新建appwidget_provider_info.xml, 名称随便, 添加以下内容


```
<appwidget-provider 
    android:initialLayout="@layout/第一步创建的布局"
    android:minHeight="xxx"
    android:minWidth="xxx"
    android:updatePeriodMills="xxx" //小部件自动更新时间, 单位为毫秒
    />
```

3. 定义小部件的实现类
> 需要继承AppWidgetProvider, 代码如下

```
public class MyWidgetProvider extends AppWidgetProvider {
    //当小部件第一次添加到桌面时调用, 多次添加也仅调用一次
    @Override
    public void onEnabled(Context context) {
        super.onEnabled(context);
    }
    //当最后一个该类型的小部件被删除时调用
    @Override
    public void onDisabled(Context context) {
        super.onDisabled(context);
    }
    //每删除一个小部件都会调用一次
    @Override
    public void onDeleted(Context context, int[] appWidgetIds) {
        super.onDeleted(context, appWidgetIds);
    }

    //小部件被添加或更新调用.更新时机由updatePeriodMillis指定.
    @Override
    public void onUpdate(Context context, AppWidgetManager appWidgetManager, int[] appWidgetIds) {
        super.onUpdate(context, appWidgetManager, appWidgetIds);
    }
}
```

4. AndroidManifest.xml中注册

```
<receiver android:name=".chapter5.MyWidgetProvider">
    <meta-data android:name="android.appwidget.provider"
        android:resource="@xml/my_widget"/>
    <intent-filter>
        <!-- 系统小部件标志 -->
        <action android:name="android.appwidget.action.APPWIDGET_UPDATE" />
    </intent-filter>
</receiver>
```

## PendingIntent概述
![image](https://raw.githubusercontent.com/weikano/NoteResources/master/2.png)
requestCode表示发送方的请求码, 会影响到flags的效果.  
flags的常见类型有:FLAG_ONE_SHOT, FLAG_NO_CREATE, FLAG_CANCEL_CURRENT和FLAG_UPDATE_CURRENT.  
PendingIntent的匹配规则, 即在什么情况下两个PendingIntent是相同的.
1. 内部Intent相同. Intent的匹配规则为ComponentName和IntentFilter相同.
2. requestCode相同.

- FLAG_ONE_SHOT
> 当前PendingIntent只能使用一次, 然后就会自动cancel掉. 如果后续还有相同的PendingIntent, 那么它们的send方法就会调用失败. 对于通知栏来说, 使用此标志, 同类的通知只能打开一次, 后续的通知单击后将无法打开.

- FLAG_NO_CREATE
> 当前描述的PendingIntentb会主动创建, 如果当前的PendingIntent之前不存在, 那么getActivity/Service/Broadcast会直接返回null. 

- FLAG_CANCEL_CURRENT
> 当前描述的PendingIntent如果存在, 那么会被cancel掉, 然后系统会创建一个新的PendingIntent. 

- FLAG_UPDATE_CURRENT
> 当前描述的PendingIntent如果存在, 那么它们都会被更新, 即Intent里面的extras会被替换.

- 如果notify方法的id是常量, 那么不管PendingIntent是否相同, 都替换前面的通知.
- 如果notify方法的id不同, 那么当PendingIntent不匹配时, 不管采用何种flag, 都不会相互干扰. 如果PendingIntent相同:
1. 如果FLAG_ONE_SHOT, 那么后续通知中的PendingIntent会和第一条通知完全一致, 包括extras, 单击任何一条通知后, 剩下的通知均无法再打开, 当所有通知被清除后, 会再次重复这个过程. 
2. 如果FLAG_CANCEN_CURRENT, 那么只有最新的通知可以打开, 之前弹出的所有通知都无法打开. 
3. 如果FLAG_UPDATE_CURRENT, 那么之前弹出的通知中的PendingIntent会被更新, 最终和最新的一条保持一致, 包括extras, 并且这些都是可以打开的.

# RmoteViews的内部机制
RemoteViews会通过Binder传递到systemserver进程, 系统会根据RemoteViews中的包名和布局信息去得到该应用的资源. 然后通过LayoutInflater加载RemoteViews中的布局文件, systemserver进程中加载后的布局是一个普通的View, 只不过对于我们的进程是一个RemoteViews. 接着系统会对View执行一系列界面更新任务, 这些人物就是通过set方法提交的, 具体的执行时机要等到RemoteViews被加载后才能执行.  
  
    
从理论上来说, 系统完全可以通过Binder支持所有的View操作, 但是View的方法太多, 而且大量的IPC操作会影响效率. 为了解决这个问题, 系统并没用通过Binder去直接支持View的跨进程访问, 而是提供了Action概念. 系统首先将View操作封装到action并跨进程传输到systemserver, 接着执行action. 在我们每一次调用set方法, RemoteViews就会添加一个对应的Action, 当我们通过NotificationManager和AppWidgetManager提交更新时, 这些Action对象就会传输到systemserver进程并依次执行. systemserver进程通过RemoteViews.apply方法进行更新.  
  
  
reply和reApply的区别在于apply会加载布局并更新界面, 而reApply仅仅更新界面.  
setOnClickPendingIntent, setPendingIntentTemplate和setOnClickFillInIntent区别:
1. setOnClickPendingIntent用于给View设置单击事件.
2. 后两者用于给集合(ListView和StackView)设置item click.

# RemoteViews的意义
