## 1.字符串

### 1.1 返回字符串类型

**c/c++代码**
- 全局变量

```
char retp[1024];
const char* getStr1(int a, int b){
    memset(retp, 0, 1024);
    char outstr[256];
    memset(outstr, 0, 256);
    if (outstr != 0){
        sprintf_s(outstr, "汉字out DLL汉字: %d + %d==%d\n", a, b, (a + b));
    }
    strcpy_s(retp, outstr);
    return retp;
}
```

- malloc函数分配空间

```
const char* getStr2(int a, int b){
    char* retp = (char*)malloc(1024);
    memset(retp, 0, 1024);
    char outstr[256];
    memset(outstr, 0, 256);
    if (outstr != 0){
        sprintf_s(outstr, "汉字out DLL汉字: %d + %d==%d\n", a, b, (a + b));
        //printf("%s",outstr);
    }
    strcpy_s(retp, strlen(outstr) + 1, outstr);
    return retp;
}
```

**java代码**

```
package com.jnademo;

import com.sun.jna.Library;
import com.sun.jna.Native;

public class JnaTest {

	public interface CLibrary extends Library {
		CLibrary INSTANCE = (CLibrary) Native.load("E:\\dllws\\jnaTest\\x64\\Debug\\jnaTest.dll",
				CLibrary.class);
		
	    String getStr1(int a, int b);
	    
	    String getStr2(int a, int b);
	}
	
	public static void main(String[] args) {
		System.setProperty("jna.encoding","GBK");
		System.out.println(CLibrary.INSTANCE.getStr1(1, 3));
		System.out.println(CLibrary.INSTANCE.getStr2(2, 3));
	}
  
}
```
要注意添加`System.setProperty("jna.encoding","GBK");`否则会出现乱码。具体规则

- c++ char* GBK编码时
```
System.setProperty("jna.encoding", "GBK");
```

- c++ char* UTF8编码时
```
System.setProperty("jna.encoding", "UTF-8");
```

另外，其实还有个更简单的办法，JNA提供了一个宽字符字符串WString，当然c++接口参数类型要使用`wchar_t*`，这样WString就可以无缝转`wchar_t*`了，不用做任何修改，也绝对不会乱码。

### 1.2 C/C++接收字符串类型

**C/C++代码**

```
bool JavaStr(char* szText, int textLen) {
    if (szText == NULL || textLen <= 0) {
        return false;
    }
    std::string strText(szText, textLen);
    //OutputDebugStringA("JavaStr:");
    //OutputDebugStringA(strText.c_str());
    cout << strText.c_str() << endl;
    return true;
}
```

**java代码**

```
package com.jnademo;

import java.io.UnsupportedEncodingException;
import java.util.Arrays;
import java.util.List;

import com.sun.jna.Library;
import com.sun.jna.Native;

public class TestStructDemo {
	
	public interface TestStruct extends Library {
		
		TestStruct INSTANCE = (TestStruct) Native.load("E:\\dllws\\jnaTest\\x64\\Debug\\jnaTest.dll",
				TestStruct.class);
		
	    public boolean JavaStr(String str, int strLen);   // 使用String传参数，中文会参数乱码
	    
	    public boolean JavaStr(byte[] str, int strLen);   // 使用byte[]传参数，中文正常
	}
    
	public static void main(String[] args) throws UnsupportedEncodingException {
        //https://blog.51cto.com/softo/6009271
        String utf8Str = "UTF8转成GBK";
        String gbkStr = new String(utf8Str.getBytes("UTF-8"), "GBK"); // UTF8转成GBK
        byte[] gbkBytes = utf8Str.getBytes("UTF-8");
        TestStruct.INSTANCE.JavaStr(gbkStr, gbkStr.length());      // 传字符串C++接收时中文是乱码
        TestStruct.INSTANCE.JavaStr(gbkBytes, gbkBytes.length);    // 传字节数组C++接收中文正常
	}

}
```

# 2. 结构体

**C/C++代码**

```
#define JNA_LIBAPI extern "C" __declspec( dllexport ) 

	//C++结构体
	struct UserStruct{
		long id;
		char* name;
		int age;
	};

	JNA_LIBAPI void sayUser(UserStruct pUserStruct);

	JNA_LIBAPI void sayUser2(UserStruct* pUserStruct);
```
定义一个结构体UserStruct，在定义两个方法，参数分别是UserStruct对象类型和UserStruct指针类型。

**java代码**

```
import java.io.UnsupportedEncodingException;
import java.util.Arrays;
import java.util.List;

import com.sun.jna.Library;
import com.sun.jna.Native;
import com.sun.jna.NativeLong;
import com.sun.jna.Structure;

public class TestStructDemo {
	
	public interface TestStruct extends Library {
		
		TestStruct INSTANCE = (TestStruct) Native.load("E:\\dllws\\jnaTest\\x64\\Debug\\jnaTest.dll", TestStruct.class);
		
		//结构体定义
	    public static class myStructure extends Structure {
	        public NativeLong id;
	        public String name;
	        public int age;
	        public static class ByReference extends myStructure implements Structure.ByReference {}
	        public static class ByValue extends myStructure implements Structure.ByValue {}
	
	        @Override
	        protected List<String> getFieldOrder() {
	            return Arrays.asList(new String[] {"id", "name", "age"});
	        }
	    }
	    //申明C端调用结构体的函数
	    public void sayUser(myStructure.ByValue struct);
	    
	    public void sayUser2(myStructure.ByReference struct);
	    
	}
    
	public static void main(String[] args) throws UnsupportedEncodingException {
		//ByValue这个类代表结构体本身
		TestStruct.myStructure.ByValue testReference = new TestStruct.myStructure.ByValue();
        testReference.age = 20;
        testReference.id = new NativeLong(10);
        testReference.name = new String("Tony");
        TestStruct.INSTANCE.sayUser(testReference);
        //ByReference代表结构体指针
        TestStruct.myStructure.ByReference testReference2 = new TestStruct.myStructure.ByReference();
        testReference2.age = 80;
        testReference2.id = new NativeLong(100);
        testReference2.name = new String("Jack");
        TestStruct.INSTANCE.sayUser2(testReference2);
	}

}
```

在Java中实现对C结构体的模拟，需要继承Structure类，利用这个类来模拟C语言的结构体。必须注意，Structure子类中公共字段的顺序，必须与C语言中结构体的顺序一致，否则会报错！因为Java调用动态库中的C函数，实际上是一段内存作为函数参数传递给C函数。动态库以为这个参数就是C语言传过来的参数。同时，C语言的结构体是一个严格规范，它定义了内存的次序，因此，Jna中模拟的结构体变量顺序绝不能错。

如果一个struct有两个int变量，`int a 和 int b` 如果Jna中的次序和C语言中次序相反，那么不会报错，但是数据将被传递到错误的字段中去。

Structure类代表了一个原生结构体。当Structure对象作为一个函数的参数或者返回值传递时，它代表结构体指针。当它被用在另一个结构体内部作为一个字段时，它代表结构体本身。

Structure类有两个内部接口Structure.ByReference和Structure.ByValue。这两个接口仅仅是标记，如果一个类实现Structure.ByReference接口，就表示这个类代表结构体指针；如果一个类实现Structure.ByValue接口，就表示这个类代表结构体本身。使用这两个接口的实现类，可以明确定义Structure实例表示的是**结构体**还是**结构体指针**。

# 3. 附录

jna 加载动态库以及函数调用例子，实际项目可能很复杂。整个项目可能用到jna的回调，结构体，结构体数组等复杂的使用方式，刚开始使用jna会搞不清C++的参数类型与jna参数类型的转换，下面是工作中总结出来的映射关系：
- **入参：就是java传递数据给C++**
- **出参：就是java接收C++传递的数据**

![](https://files.mdnice.com/user/34714/7856bc06-bf40-484e-81f8-aa71ba04825e.png)

![](https://files.mdnice.com/user/34714/1cb6ef04-b71a-4204-82ae-584eb44db942.png)

```
参考：
https://blog.csdn.net/redchairman/article/details/108438202
https://blog.csdn.net/houmingyang/article/details/127071298
https://zhuanlan.zhihu.com/p/466863639
https://blog.csdn.net/q276250281/article/details/122110681
```