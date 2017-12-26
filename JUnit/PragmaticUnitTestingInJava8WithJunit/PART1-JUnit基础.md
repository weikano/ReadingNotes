# JUnit4
不同于JUnit3, JUnit4不再使用TestCase作为测试类的parent, 任何Test类都不需要继承TestCase, **只需要在测试方法上标记@Test.**  
  
  
## 基础
- **对于每个测试方法, JUnit都会生成一个实例, 避免测试对象共享产生影响.**
- **setUp()和tearDown()方法现在被@Before和@After取代, 允许存在多个@Before和@After标识的方法, 但是执行顺序不能保证.**
- **@Test标识的方法, 执行顺序也不能保证.**
- **@BeforeClass和@AfterClass标识的方法只会执行一次.**
- **如果被测试的方法会抛出异常, 那么可以使用@Test(expected = ThrowedException.class)来测试是否正确抛出.**


## Assert
> JUnit支持两种主要的assertion, JUnit典型的assertion和Hamcrest的matcher. 

- org.junit.Assert.
> 经典的assertEquals, assertTrue, assertNotNull等等. 
- org.hamcrest.CoreMatchers. 
> matcher是一个静态方法, 用来比较两个值.