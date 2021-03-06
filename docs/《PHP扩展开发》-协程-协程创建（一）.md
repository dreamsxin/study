# 协程创建（一）

[仓库地址](https://github.com/php-extension-research/study)

## 协程相关结构定义

首先，我们需要一个`PHP`可用的协程，根据[梳理一下架构](./《PHP扩展开发》-协程-梳理一下架构.md)这篇文章的内容，我们需要在`study_coroutine.h`里面来定义：

```c++
#include "php_study.h"

namespace Study
{
class PHPCoroutine
{
    static long create(zend_fcall_info_cache *fci_cache, uint32_t argc, zval *argv);
};
}
```

然后，我们还需要一个与`PHP`无关的协程数据结构，我们定义在`include/coroutine.h`下面：

```c++
#ifndef COROUTINE_H
#define COROUTINE_H

namespace Study
{
class Coroutine
{
    static long create(coroutine_func_t fn, void* args = nullptr);
};
}

#endif	/* COROUTINE_H */
```

`coroutine_func_t`是协程需要跑的函数，我们可以定义在`include/context.h`下面：

```c++
#ifndef CONTEXT_H
#define CONTEXT_H

typedef void (*coroutine_func_t)(void*);

#endif	/* CONTEXT_H */
```

它定义了一个指向函数的指针类型，返回值是`void`，参数是一个`void *`指针。

然后，我们在`include/coroutine.h`中引入这个`context.h`文件：

```c++
#include "context.h"
```

## 协程接口参数声明

OK，此时，我们需要为`PHP`脚本提供一个创建协程的接口，我们在文件`study_coroutine_util.cc`里面来完成。首先，我们需要确定一下这个接口的参数是什么，很显然，是一个`PHP`函数：

```c++
#include "study_coroutine.h"

ZEND_BEGIN_ARG_INFO_EX(arginfo_study_coroutine_create, 0, 0, 1)
    ZEND_ARG_CALLABLE_INFO(0, func, 0)
ZEND_END_ARG_INFO()
```

其中`ZEND_BEGIN_ARG_INFO_EX`和`ZEND_END_ARG_INFO`是一对宏，用来声明函数接受的参数。其中`ZEND_BEGIN_ARG_INFO_EX`展开如下：

```c++
#define ZEND_BEGIN_ARG_INFO_EX(name, _unused, return_reference, required_num_args)	\
	static const zend_internal_arg_info name[] = { \
		{ (const char*)(zend_uintptr_t)(required_num_args), 0, return_reference, 0 },
```

`name`是参数的的名字。

`_unused`我们不用管，因为`ZEND_BEGIN_ARG_INFO_EX`展开之后，并没有用到`_unused`。

`return_reference`表示是否返回引用。

`required_num_args`表示这个函数最少需要传递的参数个数。

`ZEND_ARG_CALLABLE_INFO`用来显式声明参数为`callable`，将检查函数、成员方法是否可调用。

`ZEND_END_ARG_INFO`展开如下：

```c++
#define ZEND_END_ARG_INFO()		};
```

因此，我们对这个参数声明展开后，会得到如下内容：

```c++
ZEND_BEGIN_ARG_INFO_EX(arginfo_study_coroutine_create, 0, 0, 1)
    ZEND_ARG_CALLABLE_INFO(0, func, 0)
ZEND_END_ARG_INFO()
  
static const zend_internal_arg_info arginfo_study_coroutine_create[] = { \
		{ (const char*)(zend_uintptr_t)(1), 0, 0, 0 },
    ZEND_ARG_CALLABLE_INFO(0, func, 0)
};
```

所以，虽然我在

```c++
ZEND_BEGIN_ARG_INFO_EX(arginfo_study_coroutine_create, 0, 0, 1)
```

中写下了单词`arginfo`、`study_coroutine_create`，似乎一定是要这样写，但是，我们把`ZEND_BEGIN_ARG_INFO_EX`宏展开之后会发现，`arginfo_study_coroutine_create`只是一个变量名而已。因此，我这里这样组合这个变量是为了可读性更好，并不是一定要这样声明这个参数，这一点大家需要去注意。

## 协程接口方法声明

然后，我们需要在文件`study_coroutine_util.cc`里面去声明这个方法：

```c++
static PHP_METHOD(study_coroutine_util, create);
```

`PHP_METHOD`展开之后的内容如下：

```
#define PHP_METHOD  			ZEND_METHOD
#define ZEND_METHOD(classname, name)	ZEND_NAMED_FUNCTION(ZEND_MN(classname##_##name))
#define ZEND_MN(name) zim_##name
#define ZEND_NAMED_FUNCTION(name)		void ZEND_FASTCALL name(INTERNAL_FUNCTION_PARAMETERS)#define INTERNAL_FUNCTION_PARAMETERS zend_execute_data *execute_data, zval *return_value
```

所以，接口方法展开的内容如下：

```c++
PHP_METHOD(study_coroutine_util, create);
ZEND_METHOD(study_coroutine_util, create);
ZEND_NAMED_FUNCTION(ZEND_MN(study_coroutine_util##_##create));
ZEND_NAMED_FUNCTION(zim_##study_coroutine_util##_##create);
void ZEND_FASTCALL zim_##study_coroutine_util##_##create(INTERNAL_FUNCTION_PARAMETERS);
void ZEND_FASTCALL zim_##study_coroutine_util##_##create(zend_execute_data *execute_data, zval *return_value);
```

`static PHP_METHOD(study_coroutine_util, create);`相当于声明了如下函数：

```c++
void zim_study_coroutine_util_create(zend_execute_data *execute_data, zval *return_value);
```

（其中，`zim`是`zend internal method`的缩写）

通过对接口方法的展开，我们发现，虽然接口命名是单词`study_coroutine_util`和`create`，似乎必须得是真正的类名加上方法名。其实不然，这里也只是为了可读性更好。

我们还可以对比一下`PHP_FUNCTION`这个宏，实际上，它和`PHP_METHOD`的一个区别就是少拼接了`classname`。

## 协程接口实现

我们在`study_coroutine_util.cc`文件里面写下：

```c++
PHP_METHOD(study_coroutine_util, create)
{
    php_printf("success!\n");
}
```

因为文章篇幅的原因，我们这里简单实现。

## 协程接口收集

接着，我们需要对这个方法进行收集，放在变量`study_coroutine_util_methods`里面。在`study_coroutine_util.cc`写下：

```c++
const zend_function_entry study_coroutine_util_methods[] =
{
    PHP_ME(study_coroutine_util, create, arginfo_study_coroutine_create, ZEND_ACC_PUBLIC | ZEND_ACC_STATIC)
    PHP_FE_END
};
```

这里我们使用到一个数据结构：

```c++
typedef struct _zend_function_entry {
	const char *fname;
	zif_handler handler;
	const struct _zend_internal_arg_info *arg_info;
	uint32_t num_args;
	uint32_t flags;
} zend_function_entry;
```

`fname`是函数的名字，对应的是`PHP_ME`的第二个参数，`Zend`引擎将会创建一个包含函数名`fname`的`interned zend_string`。在这里，是`create`。这个`fname`是我们可以在`PHP`脚本中使用的。而`PHP_ME`的第一个参数`study_coroutine_util`是为了拼凑出：

```c++
PHP_METHOD(study_coroutine_util, create)
```

声明的函数：

```c++
zim_study_coroutine_util_create
```

`handler`是一个函数指针，也就是该函数的主体。那么是什么样的函数指针呢？我们得看看前面的`zif_handler`：

```c++
/* zend_internal_function_handler */
typedef void (ZEND_FASTCALL *zif_handler)(INTERNAL_FUNCTION_PARAMETERS);
```

可以发现，这是一个没有返回值，参数是`INTERNAL_FUNCTION_PARAMETERS`的函数。这个其实和`PHP_METHOD`这个宏展开得到的函数声明类型是一致的。在这里，这个`handler`存放的就是函数指针`zim_study_coroutine_util_create`。

`arg_info`是这个接口方法对应的参数。可以发现，它实际上就是我们上面的那个参数展开后的类型。所以这里很明显，我们必须填写`arginfo_study_coroutine_create`，也就是我们参数展开后定义的那个变量。

`num_args`是接口方法的参数个数。可以发现，我们这里并没有填写参数的个数，实际上，这个参数的个数会通过宏`ZEND_FENTRY`来计算出来：

```c++
#define ZEND_FENTRY(zend_name, name, arg_info, flags)	{ #zend_name, name, arg_info, (uint32_t) (sizeof(arg_info)/sizeof(struct _zend_internal_arg_info)-1), flags },
```

`flags`是标志。例如这个接口方法是`public`、`static`的。

## 协程类注册

然后，我们需要去注册我们的`Study\Coroutine`这个类。我们在`MINIT`这个阶段进行注册，代码如下：

```c++
zend_class_entry study_coroutine_ce;
zend_class_entry *study_coroutine_ce_ptr;

PHP_MINIT_FUNCTION(study)
{
	INIT_NS_CLASS_ENTRY(study_coroutine_ce, "Study", "Coroutine", study_coroutine_util_methods);

	return SUCCESS;
}
```

但是，考虑到以后我们会有许多的类，我们不在`MINIT`里面直接写注册的代码，而是让`study_coroutine_util.cc`提供一个函数，我们在这个函数里面实现注册功能：

```c++
/**
 * Define zend class entry
 */
zend_class_entry study_coroutine_ce;
zend_class_entry *study_coroutine_ce_ptr;

void study_coroutine_util_init()
{
	INIT_NS_CLASS_ENTRY(study_coroutine_ce, "Study", "Coroutine", study_coroutine_util_methods);
  study_coroutine_ce_ptr = zend_register_internal_class(&study_coroutine_ce TSRMLS_CC); // Registered in the Zend Engine
}
```

然后，我们在`php_study.h`里面来进行声明：

```c++
void study_coroutine_util_init();
```

然后，我们在`MINIT`中对这个函数进行调用，完成类的注册：

```c++
PHP_MINIT_FUNCTION(study)
{
	study_coroutine_util_init();
	return SUCCESS;
}
```

## 编译测试

```shell
~/codeDir/cppCode/study # ./make.sh 
```

编写测试脚本：

```php
<?php

Study\Coroutine::create();
```

```php
~/codeDir/cppCode/study # php test.php 
success!
~/codeDir/cppCode/study # 
```

OK，到这里，我们算是完成了协程创建接口的前期工作。

[下一篇：协程创建（二）](./《PHP扩展开发》-协程-协程创建（二）.md)

