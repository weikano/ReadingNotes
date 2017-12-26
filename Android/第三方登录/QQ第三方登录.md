## QQ第三方登录

- 创建Tencent及生命周期回调

  ```java
  private Tencent api;

  @Override
  public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    api = Tencent.createInstance(Constants.QQ_APP_ID,getApplicationContext());	
    //获取本身的账户体系，获取上次保留的QQ登录返回的openId和accessToken，建议每次都调用，保留登录状态
    api.setOpenId(openId);
    api.setAccessToken(accessToken, expires);
    //自动调用QQ登录，如不需要可以注释掉
    //如果session有效，那么就直接回调onComplete等接口，如果失效，拉起QQ登录
    //if(!api.isSessionValid()){
    //  login();
    //}
  }

  @Override
  protected void onActivityResult(int requestCode, int resultCode, Intent data) {
    //在某些低端机上调用登录后，由于内存紧张导致 APP 被系统回收，登录成功后无法成功回传数据。
    //在调用 login 的 Activity 或者 Fragment 重写 onActivityResult 方法
    if (requestCode == com.tencent.connect.common.Constants.REQUEST_LOGIN) {
      Tencent.onActivityResultData(requestCode,resultCode,data,this);
    }
    super.onActivityResult(requestCode, resultCode, data);
  }
  ```

- 调用QQ/Tim登录

  ```java
  private void login() {
    //scope为应用需要获取的api权限，用,分割，暂时只需要"get_user_info"
    //IUIListener为回调
    api.login(Activity, scope, IUiListener);
    //api.logout会清空api中的accessToken和openApi信息
  }
  ```

- 回调

  ```java
  @Override
  public void onComplete(Object o) {
    Toast.makeText(this, "Complete " + o, Toast.LENGTH_SHORT).show();
    /*o 是一个jsonObject，结构如下
    *{
  "ret":0,
  "pay_token":"xxxxxxxxxxxxxxxx",
  "pf":"openmobile_android",
  "expires_in":"7776000", token有效时长
  "openid":"xxxxxxxxxxxxxxxxxxx", 唯一标识符，提交给服务器创建用户
  "pfkey":"xxxxxxxxxxxxxxxxxxx",
  "msg":"sucess",
  "access_token":"xxxxxxxxxxxxxxxxxxxxx" accessToken
  }
    */
    //保存accessToken, expires, openApi信息
    api.setAccessToken(accessToken,expiresIn);
    api.setOpenId(openId);  
  }

  @Override
  public void onError(UiError uiError) {
    Toast.makeText(this, "ERROR # " + uiError.errorCode +"," + uiError.errorMessage +"," +   uiError.errorDetail, Toast.LENGTH_SHORT).show();
  }

  @Override
  public void onCancel() {
    Toast.makeText(this, "QQ CANCEL", Toast.LENGTH_SHORT).show();
  }
  ```

- 获取用户信息

  ```java
  //sdk提供了async的方法来获取用户信息
  //容易造成回调嵌套
  private void asycnGetUserInfo() {
    new UserInfo(context, api.getQQToken()).getUserInfo(IUiListener);
  }
  //修改UserInfo类，添加sync方法
  private JSONObject syncGetUserInfo() {
    //SyncUserInfo代码如下
    return new SyncUserInfo(context, api.getQQToken()).getUserInfo();
  }
  //或者使用api.request方法
  //官方文档上有这个Constants.GRAPH_SIMPLE_USER_INFO常量，但是sdk中找不到？
  private JSONObject syncGetUserInfoOrignial() {
    return api.request(Constants.GRAPH_SIMPLE_USER_INFO, null,
                          Constants.HTTP_GET);
  }
  ```

  ```java

  public class SyncUserInfo extends UserInfo {
      public SyncUserInfo(Context context, QQToken qqToken) {
          super(context, qqToken);
      }

      public SyncUserInfo(Context context, c c, QQToken qqToken) {
          super(context, c, qqToken);
      }

      public JSONObject getUserInfo() throws HttpUtils.NetworkUnavailableException, HttpUtils.HttpStatusException, JSONException, IOException {
          return HttpUtils.request(this.b, d.a(), "user/get_simple_userinfo", a(),"GET");
      }
  }
  ```

- 解析用户信息

  ```java
  private void parseUserInfo(JSONObject jo) {
    String nickname = jo.optString("nickname");
    String gender = jo.optString("gender");
    String province = jo.optString("province");
    String city = jo.optString("city");
    String figureurl = jo.optString("firgureurl");//QQ空间头像小30x30
    String figureurl1 = jo.optString("firgureurl_1");//QQ空间头像50x50
    String figureurl2 = jo.optString("firgureurl_2");//QQ空间头像100x100
    String figureurlQQ = jo.optString("firgureurl_qq_1");//QQ头像40x40
    String figureurlQQ2 = jo.optString("firgureurl_qq_2");//QQ头像100x100
    boolean yellowVip = Integer.parseInt(jo.optString("is_yellow_vip"))!=0;
    boolean vip = Integer.parseInt(jo.optInt("vip"))!=0;
    int yellowVipLevel = Integer.parseInt(jo.optInt("yellow_vip_level"));
    int level = Integer.parseInt(jo.optInt("level"));
    boolean yellowYearVip = Integer.parseInt(jo.optString("is_yellow_year_vip"))!=0;
  }
  ```