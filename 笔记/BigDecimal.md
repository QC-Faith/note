# BigDecimal

## 简介
Java在java.math包中提供的API类BigDecimal，用来对超过16位有效位的数进行精确的运算。双精度浮点型变量double可以处理16位有效数。在实际应用中，需要对更大或者更小的数进行运算和处理。float和double只能用来做科学计算或者是工程计算，在商业计算中要用java.math.BigDecimal。BigDecimal所创建的是对象，我们不能使用传统的+、-、*、/等算术运算符直接对其对象进行数学运算，而必须调用其相对应的方法。方法中的参数也必须是BigDecimal的对象。构造器是类的特殊方法，专门用来创建对象，特别是带有参数的对象。

## 构造器描述

BigDecimal(int)    创建一个具有参数所指定**整数值**的对象。 

BigDecimal(long)   创建一个具有参数所指定**长整数值**的对象。

BigDecimal(double) 创建一个具有参数所指定**双精度值**的对象。 //不推荐使用

BigDecimal(String) 创建一个具有参数所指定以**字符串表示的数值**的对象。//**推荐使用**

**if判断**

```java
if(a.compareTo(BigDecimal.ZERO) > 0) {}
//判断 BigDecimal 类型 a 是否大于 0
```



## 方法描述

add(BigDecimal)     BigDecimal对象中的值相加，然后返回这个对象。 
subtract(BigDecimal) BigDecimal对象中的值相减，然后返回这个对象。 
multiply(BigDecimal)  BigDecimal对象中的值相乘，然后返回这个对象。 
divide(BigDecimal)   BigDecimal对象中的值相除，然后返回这个对象。 
toString()         将BigDecimal对象的数值转换成字符串。 
doubleValue()      将BigDecimal对象中的值以双精度数返回。 
floatValue()       将BigDecimal对象中的值以单精度数返回。 
longValue()       将BigDecimal对象中的值以长整数返回。 
intValue()        将BigDecimal对象中的值以整数返回。

> JDK的描述：
>
> 1、参数类型为double的构造方法的结果有一定的不可预知性。在Java中写入newBigDecimal(0.1)所创建的BigDecimal不等于 0.1（非标度值 1，其标度为 1），实际上等于0.1000000000000000055511151231257827021181583404541015625。这是因为0.1无法准确地表示为 double（或者说对于该情况，不能表示为任何有限长度的二进制小数）。这样，传入到构造方法的值不会正好等于 0.1（虽然表面上等于该值）。
>
>  2、另一方面，String 构造方法是完全可预知的：写入 newBigDecimal("0.1") 将创建一个 BigDecimal，它正好等于预期的 0.1。因此，比较而言，**通常建议优先使用String构造方法**
>
> 3、**当double必须用作BigDecimal的源时，**请使用`Double.toString(double)``转成String，然后`使用String构造方法，或使用BigDecimal的静态方法valueOf，

### 1.注意事项

如果进行==**除法**==运算的时候，**结果不能整除，有余数**，这个时候会报**java.lang.ArithmeticException:** ，要避免这个错误产生，在进行除法运算的时候，针对可能出现的小数产生的计算，必须要多传两个参数**divide(BigDecimal，保留小数点后几位小数，舍入模式)**

### 2.八种舍入模式解释如下

**2.1 UP**

```
public final static int ROUND_UP = 0;
```



定义：远离零方向舍入。

解释：始终对非零舍弃部分前面的数字加 1。注意，此舍入模式始终不会减少计算值的绝对值。

图示：

![img](http://static.oschina.net/uploads/space/2016/0506/142941_yyn6_820500.png)

示例：

| 输入数字 | 使用 UP 舍入模式将输入数字舍入为一位数 |
| -------- | -------------------------------------- |
| 5.5      | 6                                      |
| 2.5      | 3                                      |
| 1.6      | 2                                      |
| 1.1      | 2                                      |
| 1.0      | 1                                      |
| -1.0     | -1                                     |
| -1.1     | -2                                     |
| -1.6     | -2                                     |
| -2.5     | -3                                     |
| -5.5     | -6                                     |

**
**

**2.2 DOWN**

```
public final static int ROUND_DOWN = 1;
```



定义：向零方向舍入。

解释：从不对舍弃部分前面的数字加 1（即截尾）。注意，此舍入模式始终不会增加计算值的绝对值。

图示：

![img](http://static.oschina.net/uploads/space/2016/0506/143823_ehLo_820500.png)

示例：



| 输入数字 | 使用 DOWN 舍入模式将输入数字舍入为一位数 |
| -------- | ---------------------------------------- |
| 5.5      | 5                                        |
| 2.5      | 2                                        |
| 1.6      | 1                                        |
| 1.1      | 1                                        |
| 1.0      | 1                                        |
| -1.0     | -1                                       |
| -1.1     | -1                                       |
| -1.6     | -1                                       |
| -2.5     | -2                                       |
| -5.5     | -5                                       |

**
**

**2.3 CEILING**

```
public final static int ROUND_CEILING = 2;
```



定义：向正无限大方向舍入。

解释：如果结果为正，则舍入行为类似于 RoundingMode.UP；如果结果为负，则舍入行为类似于RoundingMode.DOWN。注意，此舍入模式始终不会减少计算值。

图示：

![img](http://static.oschina.net/uploads/space/2016/0506/144340_WLoo_820500.png)

示例：

| 输入数字 | 使用 DOWN 舍入模式将输入数字舍入为一位数 |
| -------- | ---------------------------------------- |
| 5.5      | 6                                        |
| 2.5      | 3                                        |
| 1.6      | 2                                        |
| 1.1      | 2                                        |
| 1.0      | 1                                        |
| -1.0     | -1                                       |
| -1.1     | -1                                       |
| -1.6     | -1                                       |
| -2.5     | -2                                       |
| -5.5     | -5                                       |

**
**

**2.4 FLOOR**

```
public final static int ROUND_FLOOR = 3;
```



定义：向负无限大方向舍入。

解释：如果结果为正，则舍入行为类似于 RoundingMode.DOWN；如果结果为负，则舍入行为类似于RoundingMode.UP。注意，此舍入模式始终不会增加计算值。

图示：

![img](http://static.oschina.net/uploads/space/2016/0506/144734_hw8X_820500.png)

示例：

| 输入数字 | 使用 DOWN 舍入模式将输入数字舍入为一位数 |
| -------- | ---------------------------------------- |
| 5.5      | 5                                        |
| 2.5      | 2                                        |
| 1.6      | 1                                        |
| 1.1      | 1                                        |
| 1.0      | 1                                        |
| -1.0     | -1                                       |
| -1.1     | -2                                       |
| -1.6     | -2                                       |
| -2.5     | -3                                       |
| -5.5     | -6                                       |

**
**

**2.5 HALF_UP （Half指的中点值，例如0.5、0.05，0.15等等）**

```
public final static int ROUND_HALF_UP = 4;
```



定义：向最接近的数字方向舍入，如果与两个相邻数字的距离相等，则向上舍入。

解释：如果被舍弃部分 >= 0.5，则舍入行为同 RoundingMode.UP；否则舍入行为同RoundingMode.DOWN。注意，此舍入模式就是通常学校里讲的四舍五入。

图示：

![img](http://static.oschina.net/uploads/space/2016/0506/145534_4rUv_820500.png) 

示例：

| 输入数字 | 使用 DOWN 舍入模式将输入数字舍入为一位数 |
| -------- | ---------------------------------------- |
| 5.5      | 6                                        |
| 2.5      | 3                                        |
| 1.6      | 2                                        |
| 1.1      | 1                                        |
| 1.0      | 1                                        |
| -1.0     | -1                                       |
| -1.1     | -1                                       |
| -1.6     | -2                                       |
| -2.5     | -3                                       |
| -5.5     | -6                                       |

**
**

**2.6 HALF_DOWN**

```
public final static int ROUND_HALF_DOWN = 5;
```



定义：向最接近的数字方向舍入，如果与两个相邻数字的距离相等，则向下舍入。

解释：如果被舍弃部分 > 0.5，则舍入行为同 RoundingMode.UP；否则舍入行为同RoundingMode.DOWN。注意，此舍入模式就是通常讲的五舍六入。

图示：

![img](http://static.oschina.net/uploads/space/2016/0506/150054_o3Sc_820500.png)

示例：

| 输入数字 | 使用 DOWN 舍入模式将输入数字舍入为一位数 |
| -------- | ---------------------------------------- |
| 5.5      | 5                                        |
| 2.5      | 2                                        |
| 1.6      | 2                                        |
| 1.1      | 1                                        |
| 1.0      | 1                                        |
| -1.0     | -1                                       |
| -1.1     | -1                                       |
| -1.6     | -2                                       |
| -2.5     | -2                                       |
| -5.5     | -5                                       |

**
**

**2.7 HALF_EVEN**

```
public final static int ROUND_HALF_EVEN = 6;
```



定义：向最接近数字方向舍入，如果与两个相邻数字的距离相等，则向相邻的偶数舍入。

解释：如果舍弃部分左边的数字为奇数，则舍入行为同 RoundingMode.HALF_UP；如果为偶数，则舍入行为同RoundingMode.HALF_DOWN。注意，在重复进行一系列计算时，根据统计学，此舍入模式可以在统计上将累加错误减到最小。此舍入模式也称为“银行家舍入法”，主要在美国使用。此舍入模式类似于 Java 中对float 和double 算法使用的舍入策略。

图示：

![img](http://static.oschina.net/uploads/space/2016/0506/150711_ZcTd_820500.png)

示例：

| 输入数字 | 使用 DOWN 舍入模式将输入数字舍入为一位数 |
| -------- | ---------------------------------------- |
| 5.5      | 6                                        |
| 2.5      | 2                                        |
| 1.6      | 2                                        |
| 1.1      | 1                                        |
| 1.0      | 1                                        |
| -1.0     | -1                                       |
| -1.1     | -1                                       |
| -1.6     | -2                                       |
| -2.5     | -2                                       |
| -5.5     | -6                                       |

**
**

**2.8 UNNECESSARY**

```
public final static int ROUND_UNNECESSARY =  7;
```



定义：用于断言请求的操作具有精确结果，因此不发生舍入。

解释：计算结果是精确的，不需要舍入，否则抛出 ArithmeticException。

示例：

| 输入数字 | 使用 DOWN 舍入模式将输入数字舍入为一位数 |
| -------- | ---------------------------------------- |
| 5.5      | 抛出 ArithmeticException                 |
| 2.5      | 抛出 ArithmeticException                 |
| 1.6      | 抛出 ArithmeticException                 |
| 1.1      | 抛出 ArithmeticException                 |
| 1.0      | 1                                        |
| -1.0     | -1                                       |
| -1.1     | 抛出 ArithmeticException                 |
| -1.6     | 抛出 ArithmeticException                 |
| -2.5     | 抛出 ArithmeticException                 |
| -5.5     | 抛出 ArithmeticException                 |