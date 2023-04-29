# 1. 创建动态链接库项目
- 创建jnaTest项目

![](https://files.mdnice.com/user/34714/d260692a-9970-4763-8616-d3d3864bf59f.png)

下一步中填写项目名称和存储的目录；然后直接创建即可

![](https://files.mdnice.com/user/34714/f4cc60d1-18b7-4af6-93d9-9cbdb6d635a8.png)

创建结果

![](https://files.mdnice.com/user/34714/f3ccd7a9-cdda-4616-83c2-2857e218d8bd.png)

# 2. 定义头文件

```
#pragma once

#ifndef JNA_TEST_H
#define JNA_TEST_H

#ifdef __cplusplus
extern "C"
{
#endif 
	__declspec(dllexport) int add(int a, int b);

	__declspec(dllexport) void welcome(char* name);

#ifdef __cplusplus
}
#endif 
#endif //JNA_TEST_H
```

**备注:** 一定要添加`__declspec(dllexport)`,否则没有导出dll方法；在java调用这个方法时提示：
```
Exception in thread "main" java.lang.Unsatisfied
LinkError: Error looking up function
```
可参考文章
```
https://zhuanlan.zhihu.com/p/50997285
```

# 3. 添加cpp文件

```
#include "pch.h"
#include "JnaTest.h"
#include <string>

int add(int a, int b) {
	return a + b;
}

void welcome(char* name) {
	std::string temp = name;
	printf_s(name);
}
```

# 4. 编写java文件

```
package com.jnademo;

import com.sun.jna.Library;
import com.sun.jna.Native;

public class JnaTest {

	public interface CLibrary extends Library {
		CLibrary INSTANCE = (CLibrary) Native.load("E:\\dllws\\jnaTest\\x64\\Debug\\jnaTest.dll",
				CLibrary.class);

		int add(int a, int b);

		void welcome(String name);
	}
	
	public static void main(String[] args) {
		int sum = CLibrary.INSTANCE.add(10, 3);
		CLibrary.INSTANCE.welcome("JNA hello world");
		System.out.println(sum);
	}
}
```

运行结果

![](https://files.mdnice.com/user/34714/1d7fa74e-fd2e-4487-98fd-3b88359f3c23.png)

# 5. 如何检查缺少的dll依赖库

在进行生产部署时，有可能出现部署的服务器缺少依赖库的情况，这种情况下需要排查具体缺少哪个依赖库，再根据具体情况安装对应的运行环境。

查看dll或exe所依赖的dll，depends家喻户晓。可惜的是depends不支持win10，使用时直接停止响应。那么在win10上有没有类似工具呢？这里推荐一款开源工具Dependencies，非常的好用。
下载地址
```
https://github.com/lucasg/Dependencies
```

使用起来很简单，运行DependenciesGui.exe，然后直接将exe或dll文件拖到窗口中即可。

![](https://files.mdnice.com/user/34714/d32f14c6-877c-4993-ab8d-63bedf7d0d0d.png)

如果发现缺少应该的dll依赖库；根据具体情况如下地址下载对应版本的`Visual C++ Redistributable`，安装即可

```
https://www.microsoft.com/zh-cn/download/details.aspx?id=48145
```

比如像下图显示就是缺少了依赖库

![](https://files.mdnice.com/user/34714/9968a569-ad5f-4402-9217-cbae8d173b07.jpg)


