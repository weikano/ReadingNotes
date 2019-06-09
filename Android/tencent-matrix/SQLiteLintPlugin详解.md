### SQLiteLintPlugin详解

#### 1.SQLiteLintPlugin核心

SQLIteLIntPlugin关键在于SQLiteLintAndroidCore的构造方法中，会调用SQLite3ProfileHooker.hook以及SQLiteLintNativeBridge.nativeInstall方法

```java
//SQLiteLintAndroidCore.java
SQLiteLintAndroidCore(xxx,xxx,xxx) {
  //SQLiteLint的静态代码块会调用SQLiteLintNativeBridge.loadLibrary()方法加载SqliteLint-lib库
  if(SQLiteLint.getSqlExecutionCallbackMode() == HOOK) {
    SQLite3ProfileHooker.hook();
  }
  SQLiteLintNativeBridge.nativeInstall(mConcernedDbPath);
}
//SQLiteLintNativeBridge.nativeInstall方法会调用com_tencent_sqlitelint_SqliteLint.cc中的Java_com_tencent_sqlitelint_SQLiteLintNativeBridge_nativeInstall方法将需要关注的数据库地址添加到native层的LintManager的Lints_映射表中，在LintManager.NotifySqlExecution中作为过滤使用
```

接下来解析SqliteLint-lib的加载过程

```c
//loader.cc
static vector<JNIModule>* g_loaders = nullptr;
jint JNI_OnLoad() {
  //如果g_loaders中的初始化了，那么调用func函数指针
  for(auto module:g_loaders) {
    if(module->init) {
      if(it->func(vm, env) != 0) {
        return -1;
      }
    }
  }
  return JNI_VERSION_1_6;
}
```

> 那么g_loaders是什么时候插入数据的？查找g_loaders->push_back发现是通过register_module_func方法，register_module_func是通过宏MODULE_INIT来调用
>
> attribute(constructor)表示方法在main之前执行，可以看做Java中的静态代码段？

```c++
//loader.h
#define MODULE_INIT(name) \
//这里是先声明对应的ModuleInit_name函数
static int ModuleInit ##name(JavaVM*vm, JNIEnv*env); \
static void __attribute__((constructor)) MODULE_INIT_##name() \
{\
	register_module_func(#name, MOduleInit_##name, 1);\
}\
static int ModuleInit_##name(JavaVM* vm, JNIEnv* env);

#define MODULE_FINI(name) \
//
	static int ModuleFini_##name(JavaVM *vm, JNIEnv *env); \
	static void __attribute__((constructor)) MODULE_FINI_##name() \
	{ \
		register_module_func(#name, ModuleFini_##name, 0); \
	} \
	static int ModuleFini_##name(JavaVM *vm, JNIEnv *env);
    
//sqlite3_profile_hook.c
MODULE_INIT(sqlite3-profile_hook) {
  //这里是实现ModuleInit_sqlite3-profile_hook
}
```

#### 2. SQLite3ProfileHooker.hook

```java
void hook() {
  nativeStartProfile();
  if(hookOpenSQLite3Profile()) {
    nativeDoHook();
  }
}
```

接下来到JNI层

```c
//sqlite3_profile_hook.cc
//只是把kStop标记设为false
jboolean Java_com_tencent_sqlitelint_util_SQLite3ProfileHooker_nativeStartProfile(){
  kStop = false;
  return true;
}

jbool Java_com_tencent_sqlitelint_util_SQLite3ProfileHooker_nativeDoHook() {
  //跟IOCanaryPlugin一样的elfhook流程
  loaded_soinfo* soinfo = elfhook_open("libandroid_runtime.so");
  elfhook_replace(soinfo, "sqlite3_profile", (void*)hooked_sqlite3_profile, (void**)&original_sqlite3_profile);
}
//替换了SQLiteLintSqlite3ProfileCallback方法指针
void* hooked_sqlite3_profile(sqlite3* db, void(*xProfile)(void*, const char*, sqlite_uint64), void* p) {
  LOGI("hooked_sqlite3_profile call");
  return original_sqlite3_profile(db, SQLiteLintSqlite3ProfileCallback, p);
}

static void SQLiteLintSqlite3ProfileCallback(void *data, const char *sql, sqlite_uint64 tm) {
  if (kStop) {
    return;
  }
  JNIEnv*env = getJNIEnv();
  if (env != nullptr) {
    SQLiteConnection* connection = static_cast<SQLiteConnection*>(data);
    //通过抛异常的方式获取当前的方法栈
    jstring extInfo = (jstring)env->CallStaticObjectMethod(kUtilClass, kMethodIDGetThrowableStack);
    char *ext_info = jstringToChars(env, extInfo);
    //调用LintManager的NotifySqlExecution
    NotifySqlExecution(connection->label, sql, tm/1000000, ext_info);
    free(ext_info);
  } else {
    LOGW("SQLiteLintSqlite3ProfileCallback env null");
  }
}
//lint_manager.cc
void NotifySqlExecution(db_path, sql, time_cost, ext_info) {
  //调用Lint.NotifySqlExecution
  it->second->NotifySqlExecution(sql, time_cost, ext_info);
}
//lint.cc
void NotifySqlExecution(sql, time_cost, ext_info) {
  //如果是sqlite系统的语句，比如explain query plan之类的
  if(env_.IsReservedSql(sql)) {
    return;
  }
  //封装成SqlInfo结构体
  queue_.push_back(sql_info);
  queue_cv_.notify_one();
  //接下来流程类似IOCanaryPlugin，会继续调用Lint.check方法
}

void Check() {
  shared_ptr<SqlInfo> sql_info;
  SqlInfo smple_sql_info;
  while(true) {
    int ret = TakeSqlInfo(sql_info);
    //忽略错误检测
    //去掉空格全部小写
    PreProcessSqlString(sql_info->sql_);
    //判断是否是以特定动词开头的语句，CRUD
    //SqlInfoProcess().Process(sql_info)，类似预处理？
    //sql_info->CopyWithoutParse(simple_sql_info)复制到simple_sql_info
    //env_.addToSqlHistory()//添加到历史记录
    //ScheduleCheckers是通过不同的Checker来做检查
    ScheduleCheckers(CheckScene::kSample, *sql_info, published_issues);
    issued_callback_(env_.GetDbPath().c_str(), *published_issues);
  }
}
```

#### 3. 各种Checker

##### 3.1 WithoutRowIdBetterChecker

> 以下情况建议使用Without Rowid：
>
> - 有非int型主键或者复合主键
> - 单个row不是太大，比如没有text或者blob类

```c
void Check() {
  vector<TableInfo> tables = env.GetTablesInfo();
  string create_sql;
  for(auto info:tables) {
    //跳过白名单中的表
    create_sql = info.create_sql_;
    ToLowerCase(create_sql);
    //如果有找到kWithoutRowIdKeyWord关键字，那么就跳过，否则检查下一步
    //即上面的两种情况
    if(IsWithoutRowIdBetter(info)) {
      //有问题Publish
    }
  }
}
```

##### 3.2 AvoidAutoIncrementChecker

> 通常来说不需要使用AUTOINCRMENT关键字来标示字段，这会增加CPU、MEMORY、IO的负载

```c
void Check() {
  vector<TableInfo> tables = env.GetTablesInfo();
  string create_sql;
  for(const auto& info:tables) {
    //如果表名在白名单里面，就跳过
    create_sql = info.create_sql_;
    ToLowerCase(create_sql);
    //然后查找kAutoIncrementKeyWord，找到了就代表有问题
  }
}
```

##### 3.3 AvoidSelectAllChecker

```c
void Check() {
  //白名单跳过
  //is_select_all_是在SqlInfoProcessor::ProcessToken中处理的，通过正则表达式？
  if(sql_info.is_select_all_) {
    //有问题
  }
}
```

##### 3.4 RedundantIndexChecker

> 检查是否有定义重复的索引

##### 3.5 PreparedStatementBetterChecker

> 每30条sql就会检测一次？