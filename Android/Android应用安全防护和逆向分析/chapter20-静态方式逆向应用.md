### 一、 smali语法
#### 1. smali指令
略
#### 2. smali语法案例分析
略
### 二、手动注入smali语句
略
### 三、ARM指令
#### 1. ARM寻址方式
- 立即数寻址
> MOV R0, #64; R0<-64
- 寄存器寻址
> ADD R0, R1, R2; R0<-R1+R2
- 寄存器间接寻址
> LDR R0,[R1]; RO<-[R1]
- 寄存器偏移寻址
> MOV R0, R2, LSL #3; RO<-R2*8 
- 寄存器变址寻址
> LDR R0, [R1, #4]; R0<-[R1+4]
- 多寄存器寻址
> LDMIA R0, {R1, R2, R3, R4}; R1<-[R0], R2<-[R0+4], R3<-[R0+8], R4<-[R0+12]
- 堆栈寻址

#### 2. ARM中的寄存器
- RO-R3
> 用于参数函数以及返回值
- R4-R6,R8,R10-R11
> 没有特殊规定，就是普通寄存器
- R7
> 栈帧指针，指向前一个保存的栈帧和链接寄存器在栈上的地址
- R9
> 操作系统保留
- R12
> IP intra-procedure scratch
- R13
> SP stack pointer
- R14
> LR link register  
- R15
> PC program counter