# Lua5.1.4代码分析(四)Lua虚拟机概述

何为”虚拟机”

在一门脚本语言中,总会有一个虚拟机,可是”虚拟机”是什么?简而言之,这里的”虚拟机”就是使用代码实现的用于模拟计算机运行的程序.

每一门脚本语言都会有自己定义的opcode(operation code,中文一般翻译为”操作码”),可以理解为这门程序自己定义的”汇编语言”.一般的编译型语言,比如C等,经过编译器编译之后生成的都是与当前硬件环境相匹配的汇编代码;而脚本型的语言,经过前端的处理之后,生成的就是opcode,再将该opcode放在这门语言的虚拟机中逐个执行.

可见,虚拟机是个中间层,它处于脚本语言前端和硬件之间的一个程序(有些虚拟机是作为单独的程序独立存在,而Lua由于是一门嵌入式的语言是附着在宿主环境中的).


![](https://wyyhzc.gitbooks.io/gifs/content/vm1-290x300.png)

有了以上的概念,下面来简单讲解在Lua中,一份Lua代码从词法分析到语法分析再到生成opcode,最后进入虚拟机执行的大体流程.

Lua的API中提供了luaL_dofile函数,它实际上是个宏,内部首先调用luaL_loadfile函数,加载Lua代码进行语法,词法分析,生成Lua虚拟机可执行的代码,再调用lua_pcall函数,执行其中的代码

```C
    #define luaL_dofile(L, fn) \
    (luaL_loadfile(L, fn) || lua_pcall(L, 0, LUA_MULTRET, 0))
```

前半部分调用luaL_loadfile函数对Lua代码进行词法和语法分析,后半部分调用lua_pcall将第一步中分析的结果(也就是opcode)到虚拟机中执行.

首先来看luaL_loadfile函数,暂时不深入其中研究它如何分析一个Lua代码文件,先看它最后输出了什么.它最终会调用f_parser函数,这是对一个Lua代码进行分析的入口函数:

```C
    static void f_parser (lua_State *L, void *ud) {
      int i;
      Proto *tf;
      Closure *cl;
      struct SParser *p = cast(struct SParser *, ud);
      int c = luaZ_lookahead(p->z);
      luaC_checkGC(L);
      tf = ((c == LUA_SIGNATURE[0]) ? luaU_undump : luaY_parser)(L, p->z,
                                                                 &p->buff, p->name);
      cl = luaF_newLclosure(L, tf->nups, hvalue(gt(L)));
      cl->l.p = tf;
      for (i = 0; i < tf->nups; i++)  /* initialize eventual upvalues */
        cl->l.upvals[i] = luaF_newupval(L);
      setclvalue(L, L->top, cl);
      incr_top(L);
    }

```

如果暂时不深究词法分析的细节,仅看这个函数对外的输出,那么可以看到:在完成词法分析之后,返回了Proto类型的指针tf,然后将其绑定在新创建的Closure指针上,初始化UpValue,最后压入Lua栈中.

不难想像,Lua词法分析之后产生的opcode等相关数据都在这个Proto类型的结构体中.

紧跟着再来看lua_pcall函数是如何将产生的opcode放入虚拟机执行的.

lua_pcall函数中,首先获取需要调用的函数指针:


```C
    c.func = L->top - (nargs+1);  /* function to be called */
```

这里的nargs是由函数参数传入的,luaL_dofile中调用lua_pcall时这里传入的参数是0,换句话说,这里得到的函数对象指针就是在f_parser函数中最后放入Lua栈的指针.
继续往下执行,走到luaD_call函数,有这一段代码:

```C
if (luaD_precall(L, func, nResults) == PCRLUA)  /* is a Lua function? */
    luaV_execute(L, 1);  /* call it */
```

进入luaV_execute函数,这里是虚拟机执行代码的主函数:

```C
    void luaV_execute (lua_State *L, int nexeccalls) {
      LClosure *cl;
      StkId base;
      TValue *k;
      const Instruction *pc;
     reentry:  /* entry point */
      lua_assert(isLua(L->ci));
      pc = L->savedpc;
      cl = &clvalue(L->ci->func)->l;
      base = L->base;
      k = cl->p->k;
      /* main loop of interpreter */
      for (;;) {
        const Instruction i = *pc++;
        StkId ra;
        if ((L->hookmask & (LUA_MASKLINE | LUA_MASKCOUNT)) &&
            (--L->hookcount == 0 || L->hookmask & LUA_MASKLINE)) {
          traceexec(L, pc);
          if (L->status == LUA_YIELD) {  /* did hook yield? */
            L->savedpc = pc - 1;
            return;
          }
          base = L->base;
        }
        /* warning!! several calls may realloc the stack and invalidate `ra' */
        ra = RA(i);
    // 以下是各种opcode的情况处理
    }
```

可以看到,这里的pc指针里存放的是虚拟机opcode代码,它最开始从L->savepc初始化而来,而L->savepc在luaD_precall中赋值:

```C
    L->savedpc = p->code;  /* starting point */
```

这里的p就是第一步f_parser中返回的Proto指针.

现在,大致的流程已经清楚了,我们来回顾一下整个流程:

    1) 函数f_parser中,对Lua代码文件的分析返回了Proto指针
    2) 函数luaD_precall中,将Lua_state的savepc指针指向1)中的Proto结构体的code指针
    3) 函数luaV_execute中,pc指针指向2)中的savepc指针,紧跟着就是一个大的循环体,依次取出其中的opcode进行执行.
    
    
![](https://wyyhzc.gitbooks.io/gifs/content/lua_VM-211x300.png)    