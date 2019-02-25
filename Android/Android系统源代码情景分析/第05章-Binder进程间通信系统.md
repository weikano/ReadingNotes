### 第05章-Binder进程间通信系统

#### 5.1 Binder驱动程序

主要包含drivers/staging/android/binder.h(c)两个源文件组成。

##### 5.1.1 基础数据结构

- binder_work：描述待处理的工作项。这些工作项有可能属于一个进程，也有可能属于一个进程中的某个线程。

  ```c++
  struct binder_work {
    struct list_head entity;//将当前结构体嵌入到一个宿主结构中
    enum {
      BINDER_WORK_TRANSACTION = 1,
      BINDER_WORK_TRANSACTION_COMPLETE,
      BINDER_WORK_NODE,
      BINDER_WORK_DEAD_BINDER,
      BINDER_WORK_DEAD_BINDER_AND_CLEAR,
      BINDER_WORK_CLEAR_DEATH_NOTIFICATION,
    } type;//工作项的内容
  };
  ```

  

- binder_node：一个Binder实体对象。每个Service组件在Binder驱动程序中都对应有一个Binder实体对象，用来描述在内核中的状态。Binder驱动程序通过强弱引用计数维护生命周期。

  ```c++
  struct binder_node {
    int debug_id;
    struct binder_work work;
    union {
      struct rb_node rb_node;//宿主进程使用一个红黑树维护它内部的所有binder对象，这是一个节点
      struct hlist_node dead_node;//如果宿主进程死亡，使用dead_node，一个全局的hash列表   
    };
    struct binder_proc* proc;//指向一个Binder实体对象的宿主进程
    struct hlist_head refs;
    int inernal_strong_refs;
    int local_weak_refs;
    int local_strong_refs;
    void _user* ptr;//两个_user指针表示用户空间中的一个Service组件，service组件内部的一个引用计数对象的地址(weakref_impl)
    void _user* cookie;//cookie指向Service组件的地址
    unsigned has_strong_ref:1;
    unsigned pending_strong_ref:1;
    unsigned has_weak_ref:1;
    unsigned peding_weak_ref:1;
    unsigned has_async_transaction:1;//是否正在处理异步事务
    unsigned accept_fds:1;
    int min_priority:8;
    struct list_head async_todo;
  };
  ```

- binder_ref_death：Service组件的死亡接收通知。

  ```c++
  struct binder_ref_death {
    struct binder_work work;
    void _user* cookie;//接收死亡通知的对象的地址
  };
  ```

- binder_ref：Binder引用对象。每个client组件在Binder驱动程序中都对应有一个Binder引用对象，用来描述在内核中的状态

  ```c++
  struct binder_ref {
    int debug_id;
    struct rb_node rb_node_desc;//句柄值。client使用句柄值通过Binder驱动访问一个Service组件
    struct rb_node rb_node_node;//红黑树节点，保存Binder引用对象
    struct hlist_node node_entity;
    struct binder_proc* proc;//Binder引用对象的宿主进程
    struct binder_node* node;//Binder引用对象所引用的Binder实体对象
    uint32_t desc;
    int strong;
    int weak;
    struct binder_ref_death* death;
  };
  ```

- binder_buffer：内核缓冲区，用来在进程间传输数据。每个使用Binder进程间通信机制的进程在Binder驱动程序中都有一个内核缓冲区列表，用来保存Binder驱动程序为它分配的内核缓冲区

  ```c++
  struct binder_buffer {
    struct list_head entry;//内核缓冲区列表的一个节点
    struct rb_node rb_node;//空闲(正在使用中)内核缓冲区的一个节点，根据free的值
    unsigned free:1;//内核缓冲区空闲则为1
    unsigned allow_user_free:1;
    unsigned async_transaction:1;
    unsigned debug_id:29;
    //内核缓冲区正在给哪一个事务以及哪一个Binder实体对象使用
    struct binder_transaction* transaction;//事务，每个事务都关联一个目标Binder对象
    struct binder_node* target_node;
    
    size_t data_size;
    size_t offsets_size;
    uint8_t data[0];
  };
  ```

- binder_proc：描述一个正在使用Binder进程间通信机制的进程。当一个进程调用函数open打开设备文件/dev/binder时，Binder驱动程序就会创建一个binder_proc，并保存到一个全局的hash列表中。

- binder_thread：Binder线程池中的一个线程

  ```c++
  struct binder_thread {
    struct binder_proc* proc;//宿主进程
    struct rb_node rb_node;//线程池使用红黑树,这是其中一个节点
    int pid;
    int looper;
    struct binder_transaction* transaction_stack;
    struct list_head todo;
    uint32_t return_error;
    uint32_t return_error2;
    wait_queue_head_t wait;
    struct binder_stats stats;
  };
  enum {
    BINDER_LOOPER_STATE_REGISTERD = 0x01,
    ENTERED = 0x02,
    EXITED = 0x04,
    INVALID = 0x08,
    WAITING = 0x10,
    NEED_RETURN = 0x20
  };
  ```

- binder_transaction：进程间通信的过程，即事务。

  ```c++
  struct binder_transaction {
    int debug_id;
    struct binder_work work;
    struct binder_thread* from;//发起事务的线程
    struct binder_transaction* from_parent;  //一个事务依赖的另一事务
    struct binder_proc* to_proc;//负责处理事务的进程
    struct binder_thread* to_thread;//负责处理事务的线程
    struct binder_transaction* to_parent;//目标线程下一个需要处理的事务
    unsigned need_reply: 1;//1是表示是同步事务，需要等待对方回复
    struct binder_buffer* buffer;//Binder驱动程序为该事务分配的一块内存缓冲区
    unsigned int code;
    unsigned int flags;
    long priority;//优先级
    long saved_priority;//修改一个线程的优先级之前，会将原来的优先级保存在此。一边处理事务后回复原来的优先级。
    uid_t sender_euid;//用户ID
  };
  ```

- binder_ptr_cookie：一个binder实体对象或者一个Service组件的死亡通知

  ```c++
  struct binder_ptr_cookie {
    void* ptr;
    void* cookie;
  };
  ```

- binder_transaction_data：进程间通信过程中所传输的数据

  ```c++
  enum transaction_flags {
    TF_ONE_WAY = 0x01,//如果TF_ONE_WAY设置为1，表示一个异步的进程间通信过程
    TF_ROOT_OBJECT = 0x02,
    TF_STATUS_CODE = 0x08,//设置为1表示data描述的数据缓冲区的内容是一个4字节的状态码
    TF_ACCEPT_FDS = 0x16,//设置为0表示不允许目标进程返回的结果数据中包含有文件描述符
  };
  
  struct binder_transaction_data {
    //目标Binder对象(ptr指向该Binder对象对应的Service组件内部的weakref_impl)或者目标Binder引用对象(handler指向Binder引用对象的句柄)
    union {
      size_t handler;
      void* ptr;
    } target;
    //使用BR_TRANSACTION向server进程发出通信请求时，指向目标Service组件的地址。其它时候无效
    void* cookie;
    //两个进程越好的通信代码
    unsigned int code;
    //参考transaction_flags
    unsigned int flags;
  	//发起通信请求的进程的PID和UID，由Binder驱动程序填写，用于识别源进程的身份
    pid_t sender_pid;
    uid_t sender_euid;
    size_t data_size;
    size_t offsets_size;
    union {
      struct {
        const void* buffer;
        const void* offsets;
      } ptr;
      //数据量小时使用buf
      uint8_f buf[0];
    } data;
  };
  ```

- flat_binder_object：

- 