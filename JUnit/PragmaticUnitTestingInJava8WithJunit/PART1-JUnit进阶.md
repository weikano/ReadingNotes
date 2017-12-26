## Right-BICEP

- **Boundary**
> 考虑边际情况.
- **Inverse Relationship**
> 考虑用相对关系来验证结果是否正确, 比如加法对减法, 乘法和除法.
- **Cross-Checking using other mean**
> 用其它方式来交叉验证结果.
- **Forcing Error Conditions**
> 考虑异常情况. 比如 内存溢出, 磁盘空间不足, 超时, 网络异常, 系统加载等.
- **Performance Characteristics**
> 考虑程序的效率. 用时间来检验. 

## Boundary Conditions: The CORRECT way
- Comformance
> 结果是不是预期的格式.
- Ordering
> 结果集是不是像预期中的排序.
- Range
> 结果是不是在合理的范围.
- Reference
> 代码有没有应用不可直接控制的外部代码? 
- Existence
> 结果是否存在, 比如non-null, nonzero, 在一个结合里面等等.
- Cardinality
> 是否有足够多的值.
- Time
> 是不是按顺序发生? 准时发生? 及时发生? 最重要的是线程同步?