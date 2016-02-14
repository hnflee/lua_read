# Lua5.1.4代码分析(十二)-Lua内部模块的注册

今天讲解Lua内模块的注册机制.其实Lua自带的模块并不多,这是Lua被诟病比较多的地方,这样的坏处是很多东西需要自己造轮子没有标准的实现(虽然可以实现为库也不那么不方便使用),好处就是Lua足够的小,毕竟它的设计目标是定位成一个嵌入式的轻量级语言的.

这部分需要的知识点不多,但是可以顺带着理解Lua中的一些概念.

但是在讲解之前,还是先来看看另一个貌似并不那么相关的函数index2adr

```c
static TValue *index2adr (lua_State *L, int idx) {
  if (idx > 0) {
    TValue *o = L->base + (idx - 1);
    api_check(L, idx <= L->ci->top - L->base);
    if (o >= L->top) return cast(TValue *, luaO_nilobject);
    else return o;
  }
  else if (idx > LUA_REGISTRYINDEX) {
    api_check(L, idx != 0 && -idx <= L->top - L->base);
    return L->top + idx;
  }
  else switch (idx) {  /* pseudo-indices */
    case LUA_REGISTRYINDEX: return registry(L);
    case LUA_ENVIRONINDEX: {
      Closure *func = curr_func(L);
      sethvalue(L, &L->env, func->c.env);
      return &L->env;
    }
    case LUA_GLOBALSINDEX: return gt(L);
    default: {
      Closure *func = curr_func(L);
      idx = LUA_GLOBALSINDEX - idx;
      return (idx <= func->c.nupvalues)
                ? &func->c.upvalue[idx-1]
                : cast(TValue *, luaO_nilobject);
    }
  }
}
```

这个函数使用的频率非常高,可见重要性很大,所以是需要好好研究的.
前面已经提到过,一个Lua函数栈由两个指针base和top来指定,base指向函数栈底,top则指向栈顶.
回到index2addr函数中,几种情况:
1.  如果索引为正,则从函数栈底为起始位置向上查找数据
2.  如果索引为负,则从函数栈顶为起始位置向下查找数据
3.  紧跟着是几种特殊的索引值,都定义了非常大的数据,由于Lua栈限定了函数的栈尺寸,所以不会有那么大的索引,大可放心使用.
索引值为LUA_REGISTRYINDEX时,则返回的是全局数据global_state的l_registry表;如果索引值为LUA_GLOBALSINDEX,则返回该Lua_State的l_gt表.这两个表暂且不解释,紧跟着后面会谈到.

回到最开始讨论的Lua内部模块加载话题来.

Lua内部所有模块的注册都在linit.c的函数luaL_openlibs中提供.可以看到的是,它依次访问一个数组,数组中定义了每个模块的模块名及相应的模块注册函数,依次调用函数就完成了模块的注册.

```c
static const luaL_Reg lualibs[] = {
  {"", luaopen_base},
  {LUA_LOADLIBNAME, luaopen_package},
  {LUA_TABLIBNAME, luaopen_table},
  {LUA_IOLIBNAME, luaopen_io},
  {LUA_OSLIBNAME, luaopen_os},
  {LUA_STRLIBNAME, luaopen_string},
  {LUA_MATHLIBNAME, luaopen_math},
  {LUA_DBLIBNAME, luaopen_debug},
  {NULL, NULL}
};

LUALIB_API void luaL_openlibs (lua_State *L) {
  const luaL_Reg *lib = lualibs;
  for (; lib->func; lib++) {
    lua_pushcfunction(L, lib->func);
    lua_pushstring(L, lib->name);
    lua_call(L, 1, 0);
  }
}
```

我没有详细的查看每个模块的注册函数,不过还是以最简单的例子来讲解,就是最常用的print函数.

由于这个函数没有前缀,因此的它所在的模块是”",也就是一个空字符串,因此它是在base模块中注册的,调用的注册函数是luaopen_base.

紧跟着继续看luaopen_base内部调用的第一个函数base_open:

```c
static void base_open (lua_State *L) {
  /* set global _G */
  lua_pushvalue(L, LUA_GLOBALSINDEX);
  lua_setglobal(L, "_G");
  /* open lib into global table */
  luaL_register(L, "_G", base_funcs);

  // ....
}
```
首先来看最前面的两句:

```c
 /* set global _G */
  lua_pushvalue(L, LUA_GLOBALSINDEX);
  lua_setglobal(L, "_G");
```
这两句首先将LUA_GLOBALSINDEX对应的值压入栈中,其次调用”lua_setglobal(L, “_G”);”,这句代码的意思是在Lua_state的l_gt表中,当查找”_G”时,查找到的是索引值为LUA_GLOBALSINDEX的表.如果觉得有点绕,可以简单这个理解,在Lua中的G表,也就是全局表,满足这个等式”_G = _G["_G"]“,也就是这个叫”_G”的表,内部有一个key为”_G”的表是指向自己的.怀疑这个结论的,可以在Lua命令行中执行print(_G)和print(_G["_G"])看看输出结果是不是一致的.

我想,Lua中要这么处理的理由是:为了让G表和处理其它表使用同样的机制.查找一个变量时,最终会一直查到G表中,这是很自然的事情;所以为了也能按照这个机制顺利的查找到自己,于是在G表中有一个同名成员指向自己.

好了,前面两句的作用已经分析完毕.其结果有两个:
1.  _G = _G["_G"]
2.  _G表的值压入函数栈中方便了下面的调用.

继续看下面的语句:
luaL_register(L, “_G”, base_funcs);
它最终会将base_funcs中的函数注册到G表中,但是里面还有些细节需要看看的.

```c
LUALIB_API void luaI_openlib (lua_State *L, const char *libname,
                              const luaL_Reg *l, int nup) {
  if (libname) {
    int size = libsize(l);
    /* check whether lib already exists */
    luaL_findtable(L, LUA_REGISTRYINDEX, "_LOADED", 1);
    lua_getfield(L, -1, libname);  /* get _LOADED[libname] */
    if (!lua_istable(L, -1)) {  /* not found? */
      lua_pop(L, 1);  /* remove previous result */
      /* try global variable (and create one if it does not exist) */
      if (luaL_findtable(L, LUA_GLOBALSINDEX, libname, size) != NULL)
        luaL_error(L, "name conflict for module " LUA_QS, libname);
      lua_pushvalue(L, -1);
      lua_setfield(L, -3, libname);  /* _LOADED[libname] = new table */
    }
    lua_remove(L, -2);  /* remove _LOADED table */
    lua_insert(L, -(nup+1));  /* move library table to below upvalues */
  }

// ...
}
```

注册这些函数之前,首先会到l_registry表的成员_LOADED表中查找该库,如果不存在则再在G表中查找这个库,不存在则创建一个表.因此,不管是lua中内部的库或者是外部使用require引用的库,都会走这个流程并最终在G表和l_registry["_LOADED"]中存放该库的表.最后,再遍历传进来的函数指针数组,完成库函数的注册.

比如,注册os.print时,首先将print函数绑定在一个函数指针上,再去l_registry["_LOADED"]和G表中查询该名为”os”的库是否存在,不存在则创建一个表,即:
G["os"] = {}

紧跟着注册print函数,即: G["os"]["print"] = 待注册的函数指针.这样,在调用lua代码os.print(1)时,首先根据”os”到G表中查找对应的表,再在这个表中查找”print”成员得到函数指针,最后完成函数的调用.

