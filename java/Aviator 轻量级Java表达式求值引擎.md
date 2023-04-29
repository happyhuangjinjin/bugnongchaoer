### 简介

Aviator是一个高性能、轻量级的java语言实现的表达式求值引擎，主要用于各种表达式的动态求值。现在已经有很多开源可用的java表达式求值引擎，为什么还需要Avaitor呢？

Aviator的设计目标是轻量级和高性能 ，相比于Groovy、JRuby的笨重，Aviator非常小，加上依赖包也才450K,不算依赖包的话只有70K；当然，Aviator的语法是受限的，它不是一门完整的语言，而只是语言的一小部分集合。

其次，Aviator的实现思路与其他轻量级的求值器很不相同，其他求值器一般都是通过解释的方式运行，而Aviator则是直接将表达式编译成Java字节码，交给JVM去执行。简单来说，Aviator的定位是介于Groovy这样的重量级脚本语言和IKExpression这样的轻量级表达式引擎之间。

![](https://files.mdnice.com/user/34714/7025b9b6-3e42-4bd8-9600-87b09ac86b09.png)


### Aviator的特性

- 支持大部分运算操作符，包括算术操作符、关系运算符、逻辑操作符、位运算符、正则匹配操作符(=~)、三元表达式?: ，并且支持操作符的优先级和括号强制优先级，具体请看后面的操作符列表
- 支持大整数和精度运算（2.3.0版本引入）
- 支持函数调用和自定义函数
- 内置支持正则表达式匹配，类似Ruby、Perl的匹配语法，并且支持类Ruby的$digit指向匹配分组
- 自动类型转换，当执行操作的时候，会自动判断操作数类型并做相应转换，无法转换即抛异常
- 支持传入变量，支持类似a.b.c的嵌套变量访问
- 函数式风格的seq库，操作集合和数组
- 性能优秀

### Aviator的限制
- 没有if else、do while等语句，没有赋值语句，仅支持逻辑表达式、算术表达式、三元表达式和正则匹配
- 不支持八进制数字字面量，仅支持十进制和十六进制数字字面量

### jar包引入
```
<dependency>
  <groupId>com.googlecode.aviator</groupId>
  <artifactId>aviator</artifactId>
  <version>3.0.1</version>
</dependency>
```

### 如何使用Aviator

Aviator有一个统一执行的入口`AviatorEvaluator`类。他t提供了一系列的静态方法。主要用到的下面两种方法：
- AviatorEvaluator.compile
- AviatorEvaluator.execute

**execute用法**

(1) 先来看一个execute的最简单示例
```
import com.googlecode.aviator.AviatorEvaluator;


public class AviatorSimpleExample {
    public static void main(String[] args) {
        Long result = (Long) AviatorEvaluator.execute("1+2+3");
        System.out.println(result);
    }
}
```

这里要注意一个问题，为什么我们的`1+2+3`计算过后，要强制转换成Long类型？

因为Aviator只支持四种数字类型（2.3.0之后的版本）：Long、Double、big int、decimal

理论上任何整数都将转换成Long（超过Long范围的，将自动转换为big int），任何浮点数都将转换为Double

以大写字母N为后缀的整数都被认为是big int，如1N,2N,9999999999999999999999N等，都是big int类型。

以大写字母M为后缀的数字都被认为是decimal，如1M,2.222M, 100000.9999M等等，都是decimal类型。

如果都是上图中，最基础的这种数字计算，肯定不可能满足各种业务场景。

(2) 下面介绍一下传入变量

```
import com.googlecode.aviator.AviatorEvaluator;

import java.util.HashMap;
import java.util.Map;

public class AviatorSimpleExample {
    public static void main(String[] args) {
        String name = "JACK";
        Map<String,Object> env = new HashMap<>();
        env.put("name", name);
        String result = (String) AviatorEvaluator.execute(" 'Hello ' + name ", env);
        System.out.println(result);
    }
}
```

```
输出结果： Hello JACK
```

(3) 介绍一下Aviator的内置函数用法，其实特别简单，只要按照函数列表中（最下方有函数列表）定义的内容，直接使用就可以

```
import com.googlecode.aviator.AviatorEvaluator;

import java.util.HashMap;
import java.util.Map;

public class AviatorSimpleExample {
    public static void main(String[] args) {
        String str = "使用Aviator";
        Map<String,Object> env = new HashMap<>();
        env.put("str",str);
        Long length = (Long)AviatorEvaluator.execute("string.length(str)",env);
        System.out.println(length);
    }
}
```

```
输出结果 ： 14
```

**compile用法**

```
import com.googlecode.aviator.AviatorEvaluator;
import com.googlecode.aviator.Expression;

import java.util.HashMap;
import java.util.Map;

public class AviatorSimpleExample {
    public static void main(String[] args) {
        String expression = "a-(b-c)>100";
        Expression compiledExp = AviatorEvaluator.compile(expression);
        Map<String, Object> env = new HashMap<>();
        env.put("a", 100.3);
        env.put("b", 45);
        env.put("c", -199.100);
        Boolean result = (Boolean) compiledExp.execute(env);
        System.out.println(result);
    }
}
```

```
输出结果 false
```

通过上面的代码片段可以看到，使用compile方法，先生成了`Expression`；最后再由`Expression.execute`，然后传入参数map，进行计算。

这么做的目的是，在我们实际使用过程中。很多情况下，我们要计算的公式都是一样的，只是每次的参数有所区别。我们可以把一个编译好的`Expression`缓存起来。这样每次可以直接获取我们之前编译的结果直接进行计算，避免`Perm区OutOfMemory`

Aviator本身自带一个全局缓存，如果决定缓存本次的编译结果，只需要

```
Expression compiledExp = AviatorEvaluator.compile(expression,true);
```

这样设置后，下一次编译同样的表达式，Aviator会自从从全局缓存中，拿出已经编译好的结果，不需要动态编译。如果需要使缓存失效，可以使用

```
AviatorEvaluator.invalidateCache(String expression);
```

### 自定义函数的使用

```
import com.googlecode.aviator.AviatorEvaluator;
import com.googlecode.aviator.Expression;
import com.googlecode.aviator.runtime.function.AbstractFunction;
import com.googlecode.aviator.runtime.function.FunctionUtils;
import com.googlecode.aviator.runtime.type.AviatorBigInt;
import com.googlecode.aviator.runtime.type.AviatorObject;

import java.util.HashMap;
import java.util.Map;

public class AviatorSimpleExample {
    public static void main(String[] args) {
        AviatorEvaluator.addFunction(new MinFunction());
        String expression = "min(a,b)";
        Expression compiledExp = AviatorEvaluator.compile(expression, true);
        Map<String, Object> env = new HashMap<>();
        env.put("a", 100.3);
        env.put("b", 45);
        Double result = (Double) compiledExp.execute(env);
        System.out.println(result);
    }

    static class MinFunction extends AbstractFunction {
        @Override public AviatorObject call(Map<String, Object> env, AviatorObject arg1, AviatorObject arg2) {
            Number left = FunctionUtils.getNumberValue(arg1, env);
            Number right = FunctionUtils.getNumberValue(arg2, env);
            return new AviatorBigInt(Math.min(left.doubleValue(), right.doubleValue()));
        }

        public String getName() {
            return "min";
        }
    }
}
````

从实际业务中使用的自定义函数来举例。我们需要对比两个数字的大小（因为实际业务中，这两个数字为多个表达式计算的结果，如果不写自定函数，只能使用？：表达式来进行计算，会显得异常凌乱）。我们定义了一个 MinFunction 继承自 AbstractFunction 实现具体的方法，返回我们想要的结果。务必要实现 getName 这个方法，用于定义我们函数在Aviator中使用的名字。在执行compile之前，先把我们的函数Add到Aviator函数列表中，后就可以使用了。

此处输出结果为
```
45.0
```

### 内置函数列表

- 操作符列表

![](https://files.mdnice.com/user/34714/f9273ad3-47ef-468c-9c92-2c477b8ef75d.png)

![](https://files.mdnice.com/user/34714/afb7842e-38d0-455e-b217-a024a29c3619.png)


- 内置函数


![](https://files.mdnice.com/user/34714/b86554f2-ba35-4d44-a0c8-7580d6ebeabc.png)

![](https://files.mdnice.com/user/34714/d96f158f-6b9f-4912-9ea6-f7b0a5e17c78.png)


![](https://files.mdnice.com/user/34714/54229d7d-ee58-416d-9dac-3edcf052cff7.png)


![](https://files.mdnice.com/user/34714/01fb63c0-1eaa-46a4-9b10-7a97307e7da3.png)

- 常量和变量

![](https://files.mdnice.com/user/34714/9f4df097-b610-40c0-aebd-4be7a24d5084.png)

```
source: //www.cnblogs.com/kaleidoscope/p/13132315.html
```