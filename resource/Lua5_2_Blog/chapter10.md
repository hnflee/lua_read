# 编译系统(2) 跳转的处理

跳转用来控制程序的指令流程。Lua使用OP_JMP指令来执行一个跳转，有关OP_JMP的详细介绍，可以参见《虚拟机指令》。跳转可以分为条件跳转和非条件跳转。非条件跳转比较简单，我们可以先从这里入手。


### **goto**



在Lua5.2中，goto和label是新加入的statement，用来执行非条件跳转。这两个statement分别在lparser.c中的gotostat和labelstat函数中被解析。上一篇中讲过，在全局数据Dyndata中，保存着一个goto列表和一个label列表，goto和label使用一个相同的数据结构Labeldesc表示。

```
    /* description of pending goto statements and label statements */  
    typedef struct Labeldesc {  
      TString *name;  /* label identifier */  
      int pc;  /* position in code */  
      int line;  /* line where it appeared */  
      lu_byte nactvar;  /* local level where it appears in current block */  
    } Labeldesc;  
```

name用来表示label的名称，用来相互查找。如果是label，pc表示这个label对应的当前函数指令集合的位置，也就是待跳转的指令位置；如果是goto，则代表为这个goto生成的OP_JMP指令的位置。nactvar代表解析此goto或者label时，函数有多少个有效的局部变量，用来在跳转时决定需要关闭哪些upvalue。

gotostat接受一个已经为之生成好了的OP_JMP指令的位置，首先通过newlabelentry为这个goto在ls->dyd->gt中生成一个Labeldesc，用来表示未处理的goto，然后调用findlabel尝试处理这个goto。findlabel会在当前的block中查找已经定义了的label。如果找到，就调用closegoto，将这个goto对应的OP_JMP指令的跳转位置设置成label的pc，并且还要决定是否需要在OP_JMP指令中关闭一些局部变量对应的upvalue；如果没有找到，这个goto对应的OP_JMP指令的跳转位置就是一个NO_JUMP，表示未决位置，等待后面再处理。

与gotostat的处理类似，labelstat在处理label时，也会首先查找已经定义但未决的goto。如果找到了goto，也要修改其OP_JMP指令的跳转位置。整个goto和label语法分析的代码比较直接，并不难理解。而这里比较晦涩的是Lua对OP_JMP指令的处理方法。对于OP_JMP的处理还会在后面的关系和逻辑运算，以及条件跳转中使用。所以，理解Lua对OP_JMP指令的处理是理解其他编译部分的基础。



### **OP_JMP**

前面讲过，由于Lua是一编编译，所以在真正生成指令时，很多东西是没法决定的。比如OP_JMP指令，要跳转的位置可能还没有定义，所以不能知道具体的跳转位置。Lua会将这些指令先生成到函数proto的指令列表中，然后记录下他们在指令列表的位置。当可以确定时，再通过这个位置找到生成好的指令，对其进行修改。这就是指令回填。

对于一个OP_JMP指令，有两个指令参数需要回填：待关闭的upvalue起始id A和跳转偏移量sBx。当通过luaK_jump函数生成一个OP_JMP指令时，这个指令的A会被初始化成0，而sBx被初始化成NO_JUMP，并返回一个int代表这个OP_JMP指令的位置。我们保存这个返回值就可以找到这个指令，对其进行回填。

在语法分析过程中(比如后面要讲到的关系和逻辑运算)，有可能会生成一系列的OP_JMP指令，他们的跳转目标是一致的，从而形成一个跳转指令集合。Lua使用了链表的方式保存这种指令集合。这个链表每个节点就是OP_JMP指令本身，使用sBx来指向链表的下一个OP_JMP指令。如果sBx为NO_JUMP，表示链表的尾节点。所以我们只需要一个指向头节点的位置，就可以遍历整个OP_JMP指令链表。lcode.c中对于OP_JMP指令的处理函数，都是基于一个跳转链表的，只不过在一些情况下，链表中只有一个节点而已。如果需要回填一个具体的跳转地址，就会遍历链表，将每个节点的跳转地址都修正到目标位置。

![](https://git.gitbook.com/raw/wyyhzc/gifs/master/1358741149_4644.png?token=d3l5aHpjOjkyODA1MGVkLTMxZDEtNDFmOS04MjY3LWU1YzdmNjU4M2U3Nw%3D%3D)

回填跳转地址可以分为两类处理：

一种是已经生成的指令位置，Lua会立即将这些跳转指令修改成目标位置。

另一种是当前位置，也就是接下来要生成的指令的位置。在处理当前位置时，Lua并没有立即修改，而是将待修改的跳转指令串接到当前FuncState中的jpc链表上。当使用luaK_code生成下一条指令时，首先会调用dischargejpc函数，将jpc链表上的所有跳转指令修改到这个位置。这等于是延迟回填。之所以这样处理，其实是为了跳转的优化。我们回头看一下luaK_jump函数，它在生成OP_JMP指令时，会将当前jpc串接到新生成的jpc上，并且将jpc清空。这个处理实际的意思是，当生成一个OP_JMP指令时，如果有其他的OP_JMP指令需要跳转到此处，其实就等于间接跳转到新生成的跳转指令的目标位置。所以这些跳转指令不用再跳转到此处，而是直接跳转到新生成的OP_JMP的目标位置，将两步跳转合并成一步。这些指令与新生成跳转指令的目标是一致的，所以可以合并成一个跳转指令集合，等待后面一起回填。

我们接下来在来回顾一下与跳转相关的指令生成api。

luaK_jump用来生成一个新的跳转指令。luaK_concat函数用来将两个链表连接起来合成一个链表。luaK_patchlist用来回填一个链表的跳转位置。luaK_patchtohere用来将一个链表准备回填到当前位置。luaK_patchclose用来回填一个链表中的A，也就是需要关闭的upvalue id，具体可以查看goto的处理。

以上是Lua处理跳转的相关内容。跳转的处理还与逻辑和关系表达式密切相关，我们会在后面表达式部分再进行详细讲解。