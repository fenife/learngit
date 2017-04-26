linux kernel 中 va_arg 等宏的解析
---
@(linux_kernel)

#### 相关主题：
1. 可变参数的实现原理
2. 可变参数的使用


#### copy from linux-0.12
```c++

/* Amount of space required in an argument list for an arg of type TYPE.
   TYPE may alternatively be an expression whose type is used.  */
   
```

#### copy from linux-2.xx
```c++
typedef int s32;
typedef s32 acpi_native_int;

#ifndef va_arg

#ifndef _VALIST
#define _VALIST
typedef char *va_list;
#endif				/* _VALIST */

/*
 * Storage alignment properties
 */
#define  _AUPBND                (sizeof (acpi_native_int) - 1)
#define  _ADNBND                (sizeof (acpi_native_int) - 1)

/*
 * Variable argument list macro definitions
 */
#define _bnd(X, bnd)            (((sizeof (X)) + (bnd)) & (~(bnd)))
#define va_start(ap, A)         (void) ((ap) = (((char *) &(A)) + (_bnd (A,_AUPBND))))
#define va_arg(ap, T)           (*(T *)(((ap) += (_bnd (T, _AUPBND))) - (_bnd (T,_ADNBND))))
#define va_end(ap)              (void) 0

#endif				/* va_arg */
```

#### 问题
1. 为什么要分两个宏`_AUPBND`、`_ADNBND`？

#### va_list
```c++
typedef char *va_list;
```

#### 堆栈字节对齐  `(linux-2.xx)`
```c++
typedef int s32;
typedef s32 acpi_native_int;

/*
 * Storage alignment properties
 */
#define  _AUPBND                (sizeof (acpi_native_int) - 1)
#define  _ADNBND                (sizeof (acpi_native_int) - 1)
```
上面的宏展开为：
```c++
#define  _AUPBND                (sizeof (int) - 1)
#define  _ADNBND                (sizeof (int) - 1)
```
在32位机下，上面的宏即：
```c++
#define  _AUPBND                3
#define  _ADNBND                3
```
即：
```c++
#define  _AUPBND                0x00000003
#define  _ADNBND                0x00000003
```

#### 类型为`X`的参数在栈中占据的字节数
```c++
#define _bnd(X, bnd)            (((sizeof (X)) + (bnd)) & (~(bnd)))
```

而下面的宏：
```c++
_bnd (T, _AUPBND)
```
展开为：
```c++
(((sizeof (T)) + (_AUPBND)) & (~(_AUPBND)))
```
`(_AUPBND)`的值为：`0x00000003`，因此：
 `(~(_AUPBND))`的值为：`0xfffffffc`，即：`1111 ... 1100`

```c++
(((sizeof (T)) + (_AUPBND)) & (~(_AUPBND)))
```
在32位机下展开为：
```c++
( ((sizeof (T)) + 3 ) & 0xfffffffc )
```
其作用为：倘若`sizeof(T)`不是`4`的整数倍，取商去余再乘以`4`，最后得到`4`的整数倍。
比如：
```c++
( ((sizeof (T))       + 3 )      & 0xfffffffc ) => 二进制
            
    sizeof (T) = 1, 1 + 3 = 4, 4 & 0xfffffffc   => (0b)100, 结果为：4
    sizeof (T) = 3, 3 + 3 = 6, 6 & 0xfffffffc   => (0b)100, 结果为：4

	sizeof (T) = 5, 5 + 3 = 8, 8 & 0xfffffffc   => (0b)1000, 结果为：8
	...
```
即：
```c++
_bnd(char, _AUPBND) => 4
_bnd(size.5, _AUPBND) => 8
```
结论：`_bnd(X, bnd)`宏计算类型为`X`的参数在栈中占据的字节数（32位机下以`4`字节对齐）


#### va_start
	初始化参数指针`ap`，亦即计算第一个可变参数的地址
```c++
/* 输入参数：
 *   @A: 第一个参数（必须有的、固定的、由此推导出后面可变参数的地址）
 * 输出参数：
 *   @ap: 第一个可变参数的地址
 */
#define va_start(ap, A)         (void) ((ap) = (((char *) &(A)) + (_bnd (A,_AUPBND))))
```
- `A`：必须是一个固定参数
- `((char *) &(A))`：取`A`的地址，并强制类型转换为`char *`
>  `char`类型占一个字节，因此`char *`指向一个字节的地址，这种类型的指针作加减操作时是以**一个字节**为单位的
>  
>  因此`((char *) &(A))`表示`A`变量地址的第一个字节（32位机中地址占`4`字节）

- `(_bnd (A,_AUPBND))`：计算固定参数`A`在栈中所占的字节数

因此：`va_start(ap, A)`表示：参数`A`地址的第一个字节 + `A`所占的字节数，即：指向下一个参数的地址的第一个字节处，亦即：
> 将函数的固定参数`A`的下一个参数（第一个可变参数）的地址赋给`ap`

#### va_arg
```c++
/* 输入参数：
 *   @ap:  当前可变参数的地址
 *   @T:   当前可变参数的类型
 * 输出参数：
 *   @ap: 下一个可变参数的地址
 *
 * 其作用有两个：
 *   1.解引用得到当前参数的值
 *   2.指向下一个参数
 */
#define va_arg(ap, T)           (*(T *)(((ap) += (_bnd (T, _AUPBND))) - (_bnd (T,_ADNBND))))
```
- `(_bnd (T, _AUPBND))`：计算类型为当前可变参数`T`在栈中占据的字节数
- `((ap) += (_bnd (T, _AUPBND)))`：`((ap) = (ap) + (_bnd (T, _AUPBND)))`，
  赋值运算符`=`右边的`(ap)`为当前可变参数的地址的第一个字节，整个表达式表示：
> 当前可变参数地址 + 当前参数所占空间字节数 赋值给`ap`，即：
> `ap`指向下一个可变参数的地址的第一个字节处
> **注意**：此处`ap`的值已经发生了变化，不再指向当前参数，而是下一个参数

- `va_arg(ap, T)`的整个表达式：先使`ap`指向下一个可变参数，然后再减去`(_bnd(T, _ADNBND)`，得到当前参数的地址，最后才是解引用得到当前参数的值
- **思考**：为什么不先解引用得到当前参数的值，再让`ap`指向下一个参数呢？
> 因为`va_arg(ap, T)`是一个宏，最终目的是为了返回当前参数的值，上面这种方式会导致这个宏返回的是下一个参数的地址，而不是当前参数的值


#### va_end
```c++
#define va_end(ap)              (void) 0
```
`ap`不再指向堆栈
