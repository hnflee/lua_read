# Lua5.1.4代码分析(十八)-pcall的实现

Lua支持两种形式的函数调用，一种对调用过程的堆栈进行保护，即使中间过程出错，也不至于让进程退出，也就是pcall，一般在使用C调用Lua写的脚本函数时，都采用pcall方式。

对比起一般的函数调用方式，pcall多做了这些事情：对函数调用前的Lua堆栈进行保护在调用完毕之后恢复，支持传入出错时的函数在调用出错时调用。

来依次看这个过程。

首先看入口函数lua_pcall：

```c
LUA_API int lua_pcall (lua_State *L, int nargs, int nresults, int errfunc) {
  struct CallS c;
  int status;
  ptrdiff_t func;
  lua_lock(L);
  api_checknelems(L, nargs+1);
  checkresults(L, nargs, nresults);
  if (errfunc == 0)
    func = 0;
  else {
    StkId o = index2adr(L, errfunc);
    api_checkvalidindex(L, o);
    func = savestack(L, o);
  }
  c.func = L->top - (nargs+1);  /* function to be called */
  c.nresults = nresults;
  status = luaD_pcall(L, f_call, &c, savestack(L, c.func), func);
  adjustresults(L, nresults);
  lua_unlock(L);
  return status;
}

```

lua_pcall函数与lua_call相比，多了第四个参数，该函数用于传入错误处理函数在Lua栈中的地址。所以第一步将根据传入的参数得到它的值在函数栈中的地址。然后根据这些参数调用函数luaD_pcall函数。

```c
int luaD_pcall (lua_State *L, Pfunc func, void *u,
                ptrdiff_t old_top, ptrdiff_t ef) {
  int status;
  unsigned short oldnCcalls = L->nCcalls;
  ptrdiff_t old_ci = saveci(L, L->ci);
  lu_byte old_allowhooks = L->allowhook;
  ptrdiff_t old_errfunc = L->errfunc;
  L->errfunc = ef;
  status = luaD_rawrunprotected(L, func, u);
  if (status != 0) {  /* an error occurred? */
    StkId oldtop = restorestack(L, old_top);
    luaF_close(L, oldtop);  /* close eventual pending closures */
    luaD_seterrorobj(L, status, oldtop);
    L->nCcalls = oldnCcalls;
    L->ci = restoreci(L, old_ci);
    L->base = L->ci->base;
    L->savedpc = L->ci->savedpc;
    L->allowhook = old_allowhooks;
    restore_stack_limit(L);
  }
  L->errfunc = old_errfunc;
  return status;
}
```

这个函数首先将一些需要保存以便以后进行错误恢复的值保存，然后调用函数luaD_rawrunprotected。

在luaD_rawrunprotected中，

```c
int luaD_rawrunprotected (lua_State *L, Pfunc f, void *ud) {
  struct lua_longjmp lj;
  lj.status = 0;
  lj.previous = L->errorJmp;  /* chain new error handler */
  L->errorJmp = &lj;
  LUAI_TRY(L, &lj,
    (*f)(L, ud);
  );
  L->errorJmp = lj.previous;  /* restore old error handler */
  return lj.status;
}
```

可以看到的是，在Lua中，涉及到这些错误恢复的数据，实际上形成一个链条关系，这个函数首先将之前的错误链保存起来。而LUAI_TRY这个宏，会根据不同的编译器进行实现，比如C++中使用的try…catch，C中使用longjmp等。

再来看看真正出错的时候是如何处理的。

```c
void luaG_errormsg (lua_State *L) {
  if (L->errfunc != 0) {  /* is there an error handling function? */
    StkId errfunc = restorestack(L, L->errfunc);
    if (!ttisfunction(errfunc)) luaD_throw(L, LUA_ERRERR);
    setobjs2s(L, L->top, L->top - 1);  /* move argument */
    setobjs2s(L, L->top - 1, errfunc);  /* push function */
    incr_top(L);
    luaD_call(L, L->top - 2, 1);  /* call it */
  }
  luaD_throw(L, LUA_ERRRUN);
}
```

首先如果之前保存的errfunc不为空，则首先从Lua栈中得到该函数，如果判断这个地址存放的不是一个函数则直接抛出错误。否则将错误参数压入栈中调用该错误处理函数。最后调用LuaD_throw函数，这个函数与前面的LUAI_TRY宏是对应的。这样就可以回到原来保存的错误恢复地点，恢复调用前的Lua栈，继续执行下去，而不是导致宿主进程退出。