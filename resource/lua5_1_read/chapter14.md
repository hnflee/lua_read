# Lua5.1.4代码分析(十四)-Lua外部模块加载机制

前面已经提到了Lua的内部模块加载机制,今天谈谈如何加载外部模块.与模块加载相关的库都在package库中提供,代码实现在loadlib.c中.

先来看Lua中与模块相关的几个函数.

1)module
在定义一个Lua模块时,第一句代码一般都是module(xxx).来看这一句背后的含义.module调用对应的c函数是loadlib.c中的函数ll_module:

```c
static int ll_module (lua_State *L) {
  const char *modname = luaL_checkstring(L, 1);
  int loaded = lua_gettop(L) + 1;  /* index of _LOADED table */
  lua_getfield(L, LUA_REGISTRYINDEX, "_LOADED");
  lua_getfield(L, loaded, modname);  /* get _LOADED[modname] */
  if (!lua_istable(L, -1)) {  /* not found? */
    lua_pop(L, 1);  /* remove previous result */
    /* try global variable (and create one if it does not exist) */
    if (luaL_findtable(L, LUA_GLOBALSINDEX, modname, 1) != NULL)
      return luaL_error(L, "name conflict for module " LUA_QS, modname);
    lua_pushvalue(L, -1);
    lua_setfield(L, loaded, modname);  /* _LOADED[modname] = new table */
  }
  /* check whether table already has a _NAME field */
  lua_getfield(L, -1, "_NAME");
  if (!lua_isnil(L, -1))  /* is table an initialized module? */
    lua_pop(L, 1);
  else {  /* no; initialize it */
    lua_pop(L, 1);
    modinit(L, modname);
  }
  lua_pushvalue(L, -1);
  setfenv(L);
  dooptions(L, loaded - 1);
  return 0;
}
```

代码的前半部分,首先根据module(XXX)中的模块名去registry["_LOADED"]表中查找,如果找不到则创建出一个新表,这个表由_G["XXX"] = registry["_LOADED"]["XXX"].
紧跟着,在modinit函数中,将这个表的成员_M,_NAME,_PACKAGE成员分别赋值.
最后,调用setfenv将该模块对应的环境置空.但是这里得回忆一下,”模块对应的环境”实际上是什么?在lua解释器在分析一个Lua文件时,最后得出的结果都会存在一个Closure中(见f_parser函数),可见一个结论:一个Lua文件,分析完毕之后对Lua而言其实就是一个Closure,也就是函数.所以,回到前面的问题,setfenv将该模块对应的环境置空就是将这个模块分析完毕之后返回的Closure对应的env环境表置空.

而如果写下的是 module(xxx,package.seeall)呢?它将会调用后面的dooptions函数并且最后调用到package.seeall对应的处理函数:

```c
static int ll_seeall (lua_State *L) {
  luaL_checktype(L, 1, LUA_TTABLE);
  if (!lua_getmetatable(L, 1)) {
    lua_createtable(L, 0, 1); /* create new metatable */
    lua_pushvalue(L, -1);
    lua_setmetatable(L, 1);
  }
  lua_pushvalue(L, LUA_GLOBALSINDEX);
  lua_setfield(L, -2, "__index");  /* mt.__index = _G */
  return 0;
}
```

上面这段函数就两个作用,一个创建该模块对应表的metatable,另外一个将meta表的__index指向_G表–也就是说,所有在该模块找不到的变量,都会去_G表中查找.

于是,前面对module函数的分析,得出的是以下几个结论:
1.  创建模块时会创建一个表,该表挂载在registry["_LOADED"],_G[模块名]下,自然而然的,该模块中的变量(函数也是一种变量)就会挂载到这个表里面.
2.  在module函数的参数中写下package.seeall将会创建该表的metatable,同时该表的__index将指向_G表.简单的说,这个模块将可以看到所有全局环境下的变量(再一次提醒,函数也是一种变量).

明白了module背后的作用,再来看看require函数.它对应的处理函数是loadlib.c中的ll_require函数,代码太多不贴在这里,简单的说做了如下几件事情:
    1.  首先在registry["_LOADED"]表中查找该库,如果已经存在则是加载过的模块,不再重复加载直接返回.
    2.  在当前环境表中查找”loaders”变量,这里存放的是所有加载器组成的数组,在Lua代码中有四个loader:

```c
static const lua_CFunction loaders[] =
    {loader_preload, loader_Lua, loader_C, loader_Croot, NULL};
```

变量里面的loader,分别使用它们对模块进行加载.如果加载的结果,在Lua栈中返回的是一个函数(前面已经提过分析完一个Lua源代码文件返回的是一个Closure),那么说明加载成功.

    3.  最后,尝试加载该模块.加载之前在Lua栈中压入一个哨兵值sentinel,如果加载完毕之后这个值没有被改动过,则说明加载完毕,将registry["_LOADED"]赋值为true表示加载成功.