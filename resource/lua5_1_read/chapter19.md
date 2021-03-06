# Lua5.1.4代码分析(十九)-Lua协程的实现

协程是个很好的东西，它能做的事情与线程相似，区别在于：协程是使用者可控的，有API给使用者来暂停和继续执行，而线程由操作系统内核控制；另外，协程也更加轻量级。这样，在遇到某些可能阻塞的操作时，可以使用暂停协程让出CPU；而当条件满足时，可以继续执行这个协程。目前在网络服务器领域，使用Lua协程最好的范例就是ngx_lua了，我自己的项目qnode也是借助Lua协程的概念：每一个qnode中的微进程底层对应一个Lua协程，这样底层的异步操作可以在使用者使用同步的方式写出来。Coool。

来看看Lua协程内部是如何实现的。

本质上，每个Lua协程其实也是对应一个LuaState指针，所以其实它内部也是一个完整的Lua虚拟机—有完整的Lua堆栈结构，函数调用栈等等等等，绝大部分之前对Lua虚拟机的分析都可以直接套用到Lua协程中。于是，由Lua虚拟机管理着这些隶属于它的协程，当需要暂停当前运行协程的时候，就保存它的运行环境，切换到别的协程继续执行。很简单的实现。

来看看相关的API。

1.  lua_newthread 创建一个Lua协程，最终会调用的API是luaE_newthread，Lua协程在Lua中也是一个独立的Lua类型数据，它的类型是LUA_TTHREAD，创建完毕之后会照例初始化Lua的栈等结构，有一点需要注意的是，调用preinit_state初始化Lua协程的时候，传入的global表指针是来自于Lua虚拟机，换句话说，任何在Lua协程修改的全局变量，也会影响到其他的Lua协程包括Lua虚拟机本身。

2.  加载一个Lua文件并且执行对于一般的Lua虚拟机，大可以直接调用luaL_dofile即可，它其实是一个宏：

```c
#define luaL_dofile(L, fn) \
        (luaL_loadfile(L, fn) || lua_pcall(L, 0, LUA_MULTRET, 0))
```

展开来也就是当调用luaL_loadfile函数完成对该Lua文件的解析，并且没有错误时，调用lua_pcall函数执行这个Lua脚本。

但是对于Lua协程而言，却不能这么做，需要调用luaL_loadfile然后再调用lua_resume函数。所以两者的区别在于lua_pcall函数和lua_resume函数。来看看lua_resume函数的实现。这个函数做的几件事情：首先查看当前Lua协程的状态对不对，然后修改计数器：

```c
L->baseCcalls = ++L->nCcalls;
```

其次调用status = luaD_rawrunprotected(L, resume, L->top – nargs);，可以看到这个保护Lua函数堆栈的调用luaD_rawrunprotected最终调用了函数resume:

```c
static void resume (lua_State *L, void *ud) {
  StkId firstArg = cast(StkId, ud);
  CallInfo *ci = L->ci;
  if (L->status == 0) {  /* start coroutine? */
    lua_assert(ci == L->base_ci && firstArg > L->base);
    if (luaD_precall(L, firstArg - 1, LUA_MULTRET) != PCRLUA)
      return;
  }
  else {  /* resuming from previous yield */
    lua_assert(L->status == LUA_YIELD);
    L->status = 0;
    if (!f_isLua(ci)) {  /* `common' yield? */
      /* finish interrupted execution of `OP_CALL' */
      lua_assert(GET_OPCODE(*((ci-1)->savedpc - 1)) == OP_CALL ||
                 GET_OPCODE(*((ci-1)->savedpc - 1)) == OP_TAILCALL);
      if (luaD_poscall(L, firstArg))  /* complete it... */
        L->top = L->ci->top;  /* and correct top if not multiple results */
    }
    else  /* yielded inside a hook: just continue its execution */
      L->base = L->ci->base;
  }
  luaV_execute(L, cast_int(L->ci - L->base_ci));
}
```

这个函数将执行Lua代码的流程划分成了几个阶段，如果调用

```c
luaD_precall(L, firstArg - 1, LUA_MULTRET) != PCRLUA
```

那么说明这次调用返回的结果小于0，可以跟进luaD_precall函数看看什么情况下会出现这样的情况：

```c
    n = (*curr_func(L)->c.f)(L);  /* do the actual call */
    lua_lock(L);
    if (n < 0)  /* yielding? */
      return PCRYIELD;
    else {
      luaD_poscall(L, L->top - n);
      return PCRC;
    }
```

继续回到resume函数中，如果之前该Lua协程的状态是YIELD，那么说明之前被中断了，则调用luaD_poscall完成这个函数的调用。
然后紧跟着调用luaV_execute继续Lua虚拟机的继续执行。

可以看到，resume函数做的事情其实有那么几件：
1.  如果调用C函数时被YIELD了，则直接返回
2.  如果之前被YIELD了，则调用luaD_poscall完成这个函数的执行，接着调用luaV_execute继续Lua虚拟机的执行。
因此，这个函数对于函数执行中可能出现的YIELD，有充分的准备和判断，因此它不像一般的pcall那样，一股脑的往下执行，而是会在出现YIELD的时候保存现场返回，在继续执行的时候恢复现场。
3.  同时，由于resume函数是由luaD_rawrunprotected进行保护调用的，即使执行出错，也不会造成整个程序的退出。

这就是Lua协程中，比一般的Lua操作过程做的更多的地方。

最后给出一个Lua协程的例子：
co.lua

```lua
print("before")
test("123")
print("after resume")
```

co.c

```c
    #include 
    #include "lua.h"
    #include "lualib.h"
    #include "lauxlib.h"

    static int panic(lua_State *state) {
      printf("PANIC: unprotected error in call to Lua API (%s)\n",
              lua_tostring(state, -1));
      return 0;
    }

    static int test(lua_State *state) {
      printf("in test\n");
      printf("yielding\n");
      return lua_yield(state, 0);
    }

    int main(int argc, char *argv[]) {
      char *name = NULL;
      name = "co.lua";
      lua_State*  L1 = NULL;
      L1 = lua_open();
      lua_atpanic(L1, panic);
      luaL_openlibs( L1 );

      lua_register(L1, "test", test);
      lua_State*  L = lua_newthread(L1);

      luaL_loadfile(L, name);
      lua_resume(L, 0);
      printf("sleeping\n");
      sleep(1);
      lua_resume(L, 0);
      printf("after resume test\n");

      return 0;
    }
```