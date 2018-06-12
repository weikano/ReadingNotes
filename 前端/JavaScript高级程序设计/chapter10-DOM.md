### 10.1 节点层次
#### 10.1.1 node类型
由于IE没有公开node类型的构造函数，因此最好将nodeType属性与数字值进行比较
#### 10.1.2 document类型
document独享是HTMLDocument(继承自Document类型)的一个实例，表示整个HTML页面。而且document对象是window对象的一个属性，因此可以作为全局对象访问。
#### 10.1.3 ELement类型
