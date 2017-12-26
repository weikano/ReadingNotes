## 使用android自带的AccountManager

### 步骤

1. 创建自己的Authenticator - 背后的大脑。
2. 创建Activities - 用户输入登录信息。
3. 创建服务 - 跟1中authenticator通信。

### 基本概念

- Authentication Token (auth-token)

  > 一个临时的令牌，用于服务器验证帐号，基本上每个请求都要附带上。

- Your authenticating server

  > 帐号服务器。用于生成帐号对应的auth-token，并附带有效期(也可以不用)。

- AccountManager

  > 管理设备上所有的帐号。应用可以从中获取auth-token。AccountManager知道如何处理各种情况下的方法，不管是打开登录/注册界面，还是从以前的请求中取出保存的auth-token。

- AccountAuthenticator

  > 处理指定Account Type的模块。AccountManager找到合适的AccountAuthenticator来执行对应Account Type的所有操作。AccountAuthenticator知道应该显示哪个activity以便用户输入登录信息，也知道怎样找到保存好的auth-token。比如，google在android上的authenticator用Google Mail Service作为通用帐号。

- AccountAuthenticatorActivity

  > 登录/注册activity的基类。这个activity负责登录或者注册帐号并且返回auth-token给调用的authenticator。

如果你的应用需要auth-token，AccountManager.getAuthToken()。

![image](http://blog.udinic.com/assets/media/images/2013-04-24-write-your-own-android-authenticator/oauth_dance1.png)

### 第一次登录

- app找AccountManager获取auth-token。
- AccountManager请求相关的AccountAuthenticator获取auth-token。
- 如果没有，显示AccountAuthenticatorActivity。
- 用户登录，服务器返回auth-token。
- AccountManager保存auth-token。
- app拿到auth-token。

### 创建Authenticator

前面提到过，Account Authenticator是用来解决履行所有账户相关的任务的类：获取保存的auth-token，展示登录界面和通过服务器处理相关的auth-token。

创建我们自己的Authenticator需要继承AbstractAccountAuthenticator并且实现一些方法。接下来关注下两个主要的方法：

####   1. addAccount

> 当用户执行登录和添加新帐号时调用。
>
> 我们需要返回一个Bundle，里面包含一个Intent去启动我们的AccountAuthenticatorActivity。这个方法可以被AccountManager.addAccount()方法调用（需要权限）或者从设置界面中的帐号中调用。
>
> ```java
> @Override
> public Bundle addAccount(AccountAuthenticatorResponse response, String accountType, String authTokenType, String[] requiredFeatures, Bundle options) throws NetworkErrorException {
>   final Intent intent = new Intent(context, QfAuthenticatorActivity.class);
>   intent.putExtra(QfAuthenticatorActivity.ARG_ACCOUNT_TYPE, accountType);
>   intent.putExtra(QfAuthenticatorActivity.ARG_AUTH_TYPE, authTokenType);
>   intent.putExtra(QfAuthenticatorActivity.ARG_IS_ADDING_NEW_ACCOUNT, true);
>   intent.putExtra(AccountManager.KEY_ACCOUNT_MANAGER_RESPONSE, response);
>   final Bundle bundle = new Bundle();
>   bundle.putParcelable(AccountManager.KEY_INTENT, intent);
>   return bundle;
> }
> ```

####   2. getAuthToken

> 获取上次成功登录保存的auth-token。如果没有，用户会被提示登录。登录成功后，请求的app会拿到期待已久的auth-token。要实现这个目的，我们需要检查AccountManager中是否有可用的auth-token，AccountManager.peekAuthToken()。日过没有，我们返回跟addAccount方法相同的结果。
>
> ```java
> @Override
> public Bundle getAuthToken(AccountAuthenticatorResponse response, Account account, String authTokenType, Bundle options) throws NetworkErrorException {
>   final AccountManager am =  AccountManager.get(context);
>   String authToken =  am.peekAuthToken(account, authTokenType);
>
>   if(TextUtils.isEmpty(authToken)) {
>     final String password = am.getPassword(account);
>     if(!TextUtils.isEmpty(password)) {
>       //如果没有获取到auth-token，使用保存的帐号和密码登录来获取auth-token
>       //                authToken = sServerAuthenticator.userSignIn(account.name, password, authTokenType);
>     }
>   }
>   if(!TextUtils.isEmpty(authToken)) {
>     final Bundle result = new Bundle();
>     result.putString(AccountManager.KEY_ACCOUNT_NAME, account.name);
>     result.putString(AccountManager.KEY_ACCOUNT_TYPE, account.type);
>     result.putString(AccountManager.KEY_AUTHTOKEN, authToken);
>     return result;
>   }
>
>   final Intent intent = new Intent(context, QfAuthenticatorActivity.class);
>   intent.putExtra(AccountManager.KEY_ACCOUNT_MANAGER_RESPONSE, response);
>   intent.putExtra(QfAuthenticatorActivity.ARG_ACCOUNT_TYPE, account.type);
>   intent.putExtra(QfAuthenticatorActivity.ARG_AUTH_TYPE, authTokenType);
>   final Bundle bundle = new Bundle();
>   bundle.putParcelable(AccountManager.KEY_INTENT, intent);
>   return bundle;
> }
> ```
>
> 如果获取到的auth-token失效了，那么你需要重新invalidate现在的auth-token，调用AccountManager.invalidateAuthToken()方法。接下来调用getAuthToken()，会尝试重新用保存的帐号密码登录，如果失败了，会进入登录界面。

### 创建登录界面

我们的AccountAuthenticatorActivity是唯一一个需要直接跟用户交互的界面。

它会展示一个登录界面，跟服务器验证登录信息，并且返回authenticator的结果。我们需要继承AccountAuthenticatorActivity的原因是因为setAccountAuthenticatorResult()方法。这个方法负责取回验证过程的结果并且返回给Authenticator。

```java
//mAuthTokenType是类似QQ或微信第三方登录中的申请权限，比如获取简单用户资料。不同的mAuthTokenType返回的authToken可能会不同
private String mAuthTokenType = "type-get_simple_user_info";
//登录
private Intent submit(){
  String account = accountEditor.getText().toString();
  String pwd = pwdEditor.getText().toString();
  //服务器登录验证，获取authToken
  
  String authToken = authenticator.userSignIn(account, pwd, mAuthTokenType);

  final Intent intent = new Intent();
  intent.putExtra(AccountManager.KEY_ACCOUNT_NAME, account);
  intent.putExtra(AccountManager.KEY_ACCOUNT_TYPE, ARG_ACCOUNT_TYPE);
  intent.putExtra(AccountManager.KEY_AUTHTOKEN, authToken);
  intent.putExtra(PARAM_USER_PASS, pwd);
  return intent;
}
//登录完成后执行
//intent为submit返回的参数
private void finishLogin(Intent intent) {
  String accountName = intent.getStringExtra(AccountManager.KEY_ACCOUNT_NAME);
  String accountPassword = intent.getStringExtra(PARAM_USER_PASS);
  final Account account = new Account(accountName, intent.getStringExtra(AccountManager.KEY_ACCOUNT_TYPE));
  if (getIntent().getBooleanExtra(ARG_IS_ADDING_NEW_ACCOUNT, false)) {
    String authtoken = intent.getStringExtra(AccountManager.KEY_AUTHTOKEN);
    String authtokenType = mAuthTokenType;  
    //创建新用户并且设置authToken，如果没有设置会导致服务器再次调用验证    
    mAccountManager.addAccountExplicitly(account, accountPassword, null);
    mAccountManager.setAuthToken(account, authtokenType, authtoken);
  } else {
    mAccountManager.setPassword(account, accountPassword);
  }
  setAccountAuthenticatorResult(intent.getExtras());
  setResult(RESULT_OK, intent);
  finish();
}
```

这个方法获取到新的auth-token并且做了下列事情：

1. 已存在帐号但是auth-token失效

   > 我们已经有一个帐号记录在AccountManager中。新的authToken会替换掉老的。但是如果用户更改密码，你需要重新设置密码。

2. 新增一个帐号

   > 创建一个新帐号时，authToken并没有立即保存到AccountManager中。它需要明确地调用AccountManager.setAuthToken()方法。**如果不这样做，AccountManager在getAuthToken()方法调用时，会尝试再此请求服务器验证用户登录信息**。

3. 备注

   > addAccountExplicitly()方法的第三个参数为user data。可以用来保存自定义的数据，比如API KEY之类的。

在这个Activity完成登录流程之后，AccountManager已经准备好了。调用setAccountAuthenticatorResult()方法返回相关信息给Authenticator。

所有流程已经准备好了，接下来我们要怎么使用它？我们必须让我们的Authenticator对所有的app可用，包括android系统的设置界面。因为我们想要它在background中运行(登录界面不一定)，使用Service是一个明确的选择。

### 创建Service

实现起来很简单。

```java
public class QfAuthenticatorService extends Service {
    private IBinder authenticator;
    @Override
    public void onCreate() {
        super.onCreate();
        authenticator = new QfAuthenticator(this).getIBinder();
    }

    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return authenticator;
    }
}
```

接下来在AndroidManifest.xml中定义好service。

```xml
<service android:name=".QfAuthenticatorService">
  <intent-filter>
    <action android:name="android.accounts.AccountAuthenticator" />
  </intent-filter>
  <meta-data android:name="android.accounts.AccountAuthenticator"
  android:resource="@xml/authenticator"/>
</service>
```

authenticator.xml的内容如下

```xml
<?xml version="1.0" encoding="utf-8"?>
<account-authenticator xmlns:android="http://schemas.android.com/apk/res/android"
    android:accountType="_QF_ACCOUNT_TYPE" //唯一标识符。如果某些app想通过我们进行验证，它需要通过这个类型来使用AccountManager
    android:icon="@mipmap/ic_authenticator"
    android:smallIcon="@mipmap/ic_authenticator"
    android:label="@string/app_name"//显示在设置-账户界面中的名字
    android:accountPreferences="@xml/prefs"//点击账户进入后的设置界面。用来展示一些特殊的设置，比如使用debug服务器、显示debug log等。可以参考google和dropbox的实现，非必须的/>
```

prefs可以展示如下信息

![image](http://blog.udinic.com/assets/media/images/2013-04-24-write-your-own-android-authenticator/account_prefs.png)

### 某些你需要知道的东西

1. 检查已存在账户的合法性。如果你想从你自己保存的帐号中获取到authToken，首先用AccountManager.getAccounts()方法来验证帐号是否存在。
2. 保存密码。AccountManager没有加密。你不能调用其他应用的peekAuthToken()方法（"caller from uid X is different than the authenticator's uid."），但是root之后可以通过adb命令做到。