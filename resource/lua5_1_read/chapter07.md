# Lua5.1.4代码分析(七)-赋值类操作

Lua中的变量,分为三种类型:Global,Local,UpVal.前面两种不需解释,第三种变量,稍微做些代码层面上的解释,比如:

```lua
    function test()
       local a = 1
       function t()
           a = 2
       end
    end
```

对函数test而言,当对变量a进行赋值的时候,会首先查找在本函数中是否有变量a的定义,如果没有发现则向更外一层的函数去查找,这个过程一直到全局查找.如果在某个更上一层的函数中找到了,那么就是这个变量对于该函数而言就是一个Upval;否则如果在全局域中找到,则是一个全局变量.

Lua中,使用栈来作为任何数据的暂存区.因此,不论是什么类型的变量,如果需要使用,都会预先在该变量所在函数的栈中预留一个位置,FuncState结构体的成员变量freereg是用来记录当前栈中可用的数据位置,每分配出一个位置用来保存变量,都会将freereg变量加1,反之则减1.而在Lua Opcode格式文档中说明的RA(),RB(),RC()等,虽然里面的R意思是寄存器,但是其实都是指的栈中的位置,比如说在Lua虚拟机的主函数luaV_execute中,要根据某条指令,取出对应的RA()值,对应的宏是:

```C
    #define RA(i)	(base+GETARG_A(i))
```

这里的base是函数的栈基址,所以可以看到,RA(i)就是从指令i中取出A部分,然后与基址相加,得到的地址仍然是函数栈上的数据位置.

所以这里的结论是:

    1.需要有一个变量,维护着一个数据在Lua函数栈中的位置,这个变量是FuncState结构体的成员变量freereg.
    2.Lua Opcode中提到的R寄存器,其实指的是函数栈上的位置.Lua并没有像PC那样真的有所谓的寄存器.

有了上面的概念,继续看在Lua中,赋值类的操作是如何进行的.
Lua在词法分析时,使用一个结构体来保存表达式相关的信息:

```C
    typedef struct expdesc {
      expkind k;
      union {
        struct { int info, aux; } s;
        lua_Number nval;
      } u;
      int t;  /* patch list of `exit when true' */
      int f;  /* patch list of `exit when false' */
    } expdesc;
```
这里的t,f可以暂且忽略不讨论,k存放的是表达式类型,u使用一个联合体存放表达式的信息,根据每种表达式各有差异,在Lua代码中,也有了相应的注释:

```C
    typedef enum {
      VVOID,	 /* no value */
      VNIL,
      VTRUE,
      VFALSE,
      VK,		/* info = index of constant in `k' */
      VKNUM,		/* nval = numerical value */
      VLOCAL,		/* info = local register */
      VUPVAL,       /* info = index of upvalue in `upvalues' */
      VGLOBAL,	    /* info = index of table; aux = index of global name in `k' */
      VINDEXED,	    /* info = table register; aux = index register (or `k') */
      VJMP,	       /* info = instruction pc */
      VRELOCABLE,      /* info = instruction pc */
      VNONRELOC,       /* info = result register */
      VCALL,	       /* info = instruction pc */
      VVARARG	       /* info = instruction pc */
    } expkind;
```

明白了这个结构体的作用,来看Lua中如何将分析得到的表达式信息,存储到Lua函数栈中,奥秘都是lcode.c的函数luaK_exp2nextreg中:

```C
    void luaK_exp2nextreg (FuncState *fs, expdesc *e) {
      luaK_dischargevars(fs, e);
      freeexp(fs, e);
      luaK_reserveregs(fs, 1);
      exp2reg(fs, e, fs->freereg - 1);
    }
```

来看第一个函数luaK_dischargevars的作用,具体的代码不贴了,它根据k中存放的表达式类型,构建出对应的opcode存放在info字段中,同时,如果是UpVal或者Global类型的变量,则要将k置为VRELOCABLE,意思是需要重新定位,理由很简单:这些变量都不是在这个函数中定义的变量,也就是不是出现在这个函数栈中的变量,所以它们的位置需要重新定位.

紧接着的函数freeexp,它的作用是判断该表达式如果不是需要重新定位的,那么就把原来占用的寄存器空间回收,说白了就是把freearg计数减1,以让这个栈位置可以被下一次利用.

跟着的luaK_reserveregs函数,用意在于预先保留一个栈位置,即将freearg加1.

最后来到函数exp2reg中,它会调用函数discharge2reg,将每种表达式类型对应的Opcode写到预留的栈位置,如果这个表达式是需要重定位的(VRELOCABLE类型),那么需要将原来的指令拿出,修改对应的寄存器位置:

```C
    case VRELOCABLE: {
          Instruction *pc = &getcode(fs, e);
          SETARG_A(*pc, reg);
          break;
        }
```

到此,涉及到最基本的赋值操作就解释完毕了.在Lua中,与之相关的Opcode有如下这些,做一个简单的解释:

```C
    OP_MOVE,/*	A B	R(A) := R(B)					*/
```

MOVE指令,用于存放在栈上的变量之间的相互赋值.

```C
    OP_LOADK,/*	A Bx	R(A) := Kst(Bx)					*/
```

从以Bx为索引,从常量数组中取出一个值赋值到A指向的栈位置.

```C
    OP_LOADBOOL,/*	A B C	R(A) := (Bool)B; if (C) pc++			*/
```

将布尔值B赋值到A指向的栈位置中,如果C为真,则PC指针加1

```C
    OP_LOADNIL,/*	A B	R(A) := ... := R(B) := nil			*/
```

将从A到B的栈数据,全部赋值为nil

```C
    OP_GETUPVAL,/*	A B	R(A) := UpValue[B]				*/
```

以B为索引从Upval数组中取值赋值给A指向的栈数据.

```C
    OP_GETGLOBAL,/*	A Bx	R(A) := Gbl[Kst(Bx)]				*/
```

以B为索引,从常量数组中取值,作为索引,再到全局数组中取值赋值给A指向的栈数据.

```C
    OP_SETGLOBAL,/*	A Bx	Gbl[Kst(Bx)] := R(A)				*/
```

参见GETGLOBAL指令,相反的操作

```C
    OP_SETUPVAL,/*	A B	UpValue[B] := R(A)				*/
```

参见GETUPVAL指令,相反的操作

