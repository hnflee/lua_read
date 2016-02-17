# Lua5.1.4代码分析(二十二)-表的赋值和查询

这里涉及到的两个OPCODE，SETTABLE和GETTABLE，由名字可以知道，分别用于表的赋值和查询。
两个OPCODE指令格式如下：
```c
OP_SETTABLE,/*  A B C R(A)[RK(B)] := RK(C)        */
OP_GETTABLE,/*  A B C R(A) := R(B)[RK(C)]       */
```

这里指令格式倒是很简单，分别用来表示表，索引和目的地址。唯一一个需要注意的是，目的地址有时候是指向寄存器的地址，有时候表示的又是常量，也就是在全局K数组中的索引。

注意在上面的格式中，有些是R()形式的，有些是RK（）形式的，R（）表示这个值仅用来表示在寄存器中的索引（R就是register的缩写），而RK()表示这个值既可以用来做为寄存器索引，也可以用来表示常量索引（在Lua中有一个全局的K数组，这里存放的都是常量，缩写K因此得名）。因此这里的重点是要了解一些RK()形式的数据表示。

Lua原码中，为RK（）形式的数据定义了一个宏，专门用来判断这类型的数据，然后自动从寄存器或者常量数组中获取数据：

```c
#define RKB(i)  check_exp(getBMode(GET_OPCODE(i)) == OpArgK, \
  ISK(GETARG_B(i)) ? k+INDEXK(GETARG_B(i)) : base+GETARG_B(i))
#define RKC(i)  check_exp(getCMode(GET_OPCODE(i)) == OpArgK, \
  ISK(GETARG_C(i)) ? k+INDEXK(GETARG_C(i)) : base+GETARG_C(i))
```

可以看到RK(B)和RK(C)做的事情差不多，差别仅在于使用一个OPCODE中取出B还是C指令域来做操作，所以这里只需要分析一个就够了。

首先，先根据getBMode判断该数据是不是OpArgK类型的，如果不是这类型的数据将报错(check_exp(A,B)表达式都是首先使用A表达式进行一些判断，不满足条件直接报错)：

```c
#define getBMode(m) (cast(enum OpArgMask, (luaP_opmodes[m] >> 4) & 3))
```

其次，ISK宏用于判断该取值是不是从数组K中取值：

```c
#define ISK(x)    ((x) & BITRK)

#define BITRK   (1 << (SIZE_B - 1))

#define SIZE_B    9
```

最后在根据这个判断的结果去K或者Lua中获取数据：

```
 k+INDEXK(GETARG_B(i)) : base+GETARG_B(i))
```

由前面可知，SIZE_B的大小为9byte，因此如果是从Lua寄存器中获取数据，最多可以到1 << (9 - 1) = 256，此时第9位不是1，因此最大可以从Lua寄存器也就是Lua栈中获取数据的索引到256为止。如果大于这个数，则需要减去256得到在K数组中的索引，比如257对应的是K数组索引中的第一个数据。这个获取K数组索引的宏是INDEXK：

```c
#define INDEXK(r) ((int)(r) & ~BITRK)
```
Lua使用这个技术，让同样的一段数据，能够在不同情况下表示两种类型的数据，在这里减少了再一次获取数据的操作。比如假如这里不能从K数组中获取数据，还得再设计一个配套的OPCODE和这两个指令配合从K数组中获取数据。
