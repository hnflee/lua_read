# 虚拟机指令(1) 概述 
 
Lua一直把虚拟机执行代码的效率作为一个非常重要的设计目标。而采用什么样的指令系统的对于虚拟机的执行效率来说至关重要。


### **Stack based vs Register based VM**



根据指令获取操作数方式的不同，我们可以把虚拟机的实现分为stack based和register based。
Stack based vm
对于大多数的虚拟机，比如JVM，Python，都采用传统的stack based vm。

Stack based vm的指令一般都是在当前stack中获取和保存操作数的。比如一个简单的加法赋值运算:a=b+c，对于stack based vm，一般会被转化成如下的指令:

```
    push b; // 将变量b的值压入stack  
    push c; // 将变量c的值压入stack  
    add;    // 将stack顶部的两个值弹出后相加，将结果压入stack  
    mov a;  // 将stack顶部结果放到a中  

```

由于Stack based vm的指令都是基于当前stack来查找操作数的，这就相当于所有操作数的存储位置都是运行期决定的，在编译器的代码生成阶段不需要额外为在哪里存储操作数费心，所以stack based的编译器实现起来相对比较简单直接。也正因为这个原因，每条指令占用的存储空间也比较小。

但是，对于一个简单的运算，stack based vm会使用过多的指令组合来完成，这样就增加了整体指令集合的长度。vm会使用同样多的迭代次数来执行这些指令，这对于效率来说会有很大的影响。并且，由于操作数都要放到stack上面，使得移动这些操作数的内存复制大大增加，这也会影响到效率。


### **Register based vm**


Lua 采用的是register based vm。

Register based vm的指令都是在已经分配好的寄存器中存取操作数。对于上面的运算，register based vm一般会使用如下的指令:

```
add a b c; // 将b与c对应的寄存器的值相加，将结果保存在a对应的寄存器中  
```

Register based vm的指令可以直接对应标准的3地址指令，用一条指令完成了上面多条指令的计算工作，并且有效地减少了内存复制操作。这样的指令系统对于效率有很大的帮助。

不过，在编译器设计上，就要在代码生成阶段对寄存器进行分配，增加了实现的复杂度。并且每条指令所占用的存储空间也相应的增加了。

### **Lua虚拟机指令简介**




Lua的指令使用一个32bit的unsigned integer表示。所有指令的定义都在lopcodes.h文件中，使用一个enum OpCode代表指令类型。在lua5.2中，总共有40种指令(id从0到39)。根据指令参数的不同，可以将所有指令分为4类:

![](https://git.gitbook.com/raw/wyyhzc/-lua5-2-/master/1357792149_6544.png?token=d3l5aHpjOjkyODA1MGVkLTMxZDEtNDFmOS04MjY3LWU1YzdmNjU4M2U3Nw%3D%3D)

除了sBx之外，所有的指令参数都是unsigned integer类型。sBx可以表示负数，但表示方法比较特殊。sBx的18bit可表示的最大整数为262143，这个数的一半131071用来表示0，所以-1可以表示为-1+131071，也就是131070，而+1可以表示为+1+131071，也就是131072。

ABC一般用来存放指令操作数据的地址，而地址可以分成3种：
* 寄存器id
* 常量表id
* upvalue id

Lua使用当前函数的stack作为寄存器使用，寄存器id从0开始。当前函数的stack与寄存器数组是相同的概念。stack(n)其实就是register(n)。

每一个函数prototype都有一个属于本函数的常量表，用于存放编译过程中函数所用到的常量。常量表可以存放nil,boolean,number和string类型的数据，id从1开始。

每一个函数prototype中都有一个upvalue描述表，用于存放在编译过程中确定的本函数所使用的upvalue的描述。在运行期，通过OP_CLOSURE指令创建一个closure时，会根据prototype中的描述，为这个closure初始化upvalue表。upvalue本身不需要使用名称，而是通过id进行访问。

A被大多数指令用来指定计算结果的目标寄存器地址。很多指令使用B或C同时存放寄存器地址和常量地址，并通过最左面的一个bit来区分。在指令生成阶段，如果B或C需要引用的常量地址超出了表示范围，则首先会生成指令将常量装载到临时寄存器，然后再将B或C改为使用该寄存器地址。

在lopcodes.h中，对于每个指令，在源码注释中都有简单的操作描述。本文接下来将针对每一个指令做更详细的描述，并给出关于这个指令的示例代码。示例代码可以帮助我们构建出一个指令使用的具体上下文，有助于进一步理解指令的作用。对指令上下文的理解还可以作为进一步研究lua的编译和代码生成系统的基础。

在分析过程中，我们使用luac来显示示例代码所生成的指令。luac的具体使用方式为:

```
luac -l -l test.lua  
```

