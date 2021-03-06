# Lua5.1.4代码分析(五)Lua栈

上一篇中,已经将Lua代码从分析到执行的大致流程分析了一遍,但仅有这些,还不足以了解Lua的运作机制,这里要分析一下Lua中的另外一个重要概念:栈.

Lua的栈在Lua中扮演着一个非常重要的中间层的角色.既然Lua虚拟机模拟的是CPU的运作,那么Lua栈模拟的就是内存的角色.在Lua内部,参数的传递是通过Lua栈,同时Lua与C等外部进行交互的时候也是使用的栈.这些暂时不展开讨论,先关注的是Lua栈的分配,管理和相关的数据结构.

lua虚拟机在初始化创建lua_State结构体时,会走到stack_init函数中,这个函数主要就是对Lua栈和CallInfo数组的初始化:

```C
    static void stack_init (lua_State *L1, lua_State *L) {
      /* initialize CallInfo array */
      L1->base_ci = luaM_newvector(L, BASIC_CI_SIZE, CallInfo);
      L1->ci = L1->base_ci;
      L1->size_ci = BASIC_CI_SIZE;
      L1->end_ci = L1->base_ci + L1->size_ci - 1;
      /* initialize stack array */
      L1->stack = luaM_newvector(L, BASIC_STACK_SIZE + EXTRA_STACK, TValue);
      L1->stacksize = BASIC_STACK_SIZE + EXTRA_STACK;
      L1->top = L1->stack;
      L1->stack_last = L1->stack+(L1->stacksize - EXTRA_STACK)-1;
      /* initialize first ci */
      L1->ci->func = L1->top;
      setnilvalue(L1->top++);  /* `function' entry for this `ci' */
      L1->base = L1->ci->base = L1->top;
      L1->ci->top = L1->top + LUA_MINSTACK;
    }

```

可以看到的是,初始化了两个数组,分别保存Lua栈和CallInfo结构体数组.
其中,与Lua栈相关的lua_State结构体成员变量有base,stack,top,lastfree,stack保存的是数组的初始位置,base会根据每次函数调用的情况发生变化,top指针指向的是当前第一个可用的栈位置,每次向栈中增加/删减元素都要对应的增减top指针,lastfee指针指向的书Lua栈的最后位置.

CallInfo结构体,是每次有函数调用时都会去初始化的一个结构体,它的成员变量中,也有top,base指针,同样的是指向Lua栈的位置,所不同的是,它关注的仅是函数调用时的相关位置.从代码中可以看出,CallInfo数组是有限制的,换言之,在Lua中的嵌套函数调用层次也是有限制,不能超过一定数量.

![](https://wyyhzc.gitbooks.io/gifs/content/struct-300x170.png)

回到上一篇中的讲解,首先看f_parser函数:

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

f_parser函数的最后两句,将分析完毕之后的结构Closure指针压入了Lua栈.

再来看luaD_precall函数,这里为将代码放入Lua虚拟机中执行准备了相关数据,我们只截取其中的一部分来看:

```C
    int luaD_precall (lua_State *L, StkId func, int nresults) {
      ….
      if (!cl->isC) {  /* Lua function? prepare its call */
        CallInfo *ci;
        StkId st, base;
        Proto *p = cl->p;
    
    // 1) 根据函数的参数类型,计算出该CallInfo的base指针位置
        if (!p->is_vararg) {  /* no varargs? */
          base = func + 1;
          if (L->top > base + p->numparams)
            L->top = base + p->numparams;
        }
        else {  /* vararg function */
          int nargs = cast_int(L->top - func) - 1;
          base = adjust_varargs(L, p, nargs);
          func = restorestack(L, funcr);  /* previous call may change the stack */
    }
    
    // 2) 分配一个新的CallInfo结构体,用于保存此次函数调用的相关信息:top,base指针,func函数
        ci = inc_ci(L);  /* now `enter' new function */
        ci->func = func;
        L->base = ci->base = base;
        ci->top = L->base + p->maxstacksize;
        lua_assert(ci->top <= L->stack_last);
    
        // 3) LuaState的PC指针指向函数原型的代码数组
        L->savedpc = p->code;  /* starting point */
    	// …..
        return PCRLUA;
      }
```

到这一步,跟某次具体的Lua代码执行相关的代码(保存在Proto的code数组中)和执行时所需环境(Lua栈),就已经准备完毕了.后面就是进入Lua虚拟机的主循环中解释执行代码了.

