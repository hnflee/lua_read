# Lua5.1.4代码分析(十五)-Lua继承机制分析

Lua并不是一门以OO为卖点的脚本语言，但是这并不妨碍在Lua中使用Lua的一些特性来实现类面向对象的特性。

先简单看看Lua中实现继承的简单示例代码，再展开分析。如下定义了两个模块，base.lua和test.lua，其中后者继承自前者：
base.lua
```lua
module( "base", package.seeall )
function new( )
    local obj = {}
    setmetatable( obj, { __index = base } )
    return obj
end
```

test.lua

```lua
module( "test", package.seeall )
setmetatable( test, {__index = base} )

function new( )
    local obj = {}
    setmetatable( obj, { __index = test } )
    return obj
end
```

1.  setmetatable的作用
setmetatable函数，在Lua自带的base模块中实现，对应的函数在luaB_setmetatable中。简单的说，这个函数所做的事情是：
首先， 检查是否已经有设置”__metatable”的metatable的被设置过，如果有则不能进行修改。
比如这样的代码：
```lua
local tbl = {}
setmetatable(tbl, {__metatable=1})
setmetatable(tbl, {})
```
会报错：”cannot change a protected metatable”.
再次，将传入的表赋值给该数据的metatable表。
2.  虚拟机如何查找表成员
来看看要访问一个表类型的数据，Lua虚拟机内部是如何进行查找的。最终会走到以下这个函数来查找：
```c
void luaV_gettable (lua_State *L, const TValue *t, TValue *key, StkId val) {
  int loop;
  for (loop = 0; loop < MAXTAGLOOP; loop++) {
    const TValue *tm;
    if (ttistable(t)) {  /* `t' is a table? */
      Table *h = hvalue(t);
      const TValue *res = luaH_get(h, key); /* do a primitive get */
      if (!ttisnil(res) ||  /* result is no nil? */
          (tm = fasttm(L, h->metatable, TM_INDEX)) == NULL) { /* or no TM? */
        setobj2s(L, val, res);
        return;
      }
      /* else will try the tag method */
    }
    else if (ttisnil(tm = luaT_gettmbyobj(L, t, TM_INDEX)))
      luaG_typeerror(L, t, "index");
    if (ttisfunction(tm)) {
      callTMres(L, val, tm, t, key);
      return;
    }
    t = tm;  /* else repeat with `tm' */
  }
  luaG_runerror(L, "loop in gettable");
}
```

简单来看看这个函数的流程：
首先，如果待查找对象是一个表，那么根据传入的key来进行第一步的查找，如果查找到了相应的结果，或者这个表的metatable表的__index字段为空，说明不必进行进一步的查找了，直接返回查找的结果：

```c
if (ttistable(t)) {  /* `t' is a table? */
      Table *h = hvalue(t);
      const TValue *res = luaH_get(h, key); /* do a primitive get */
      if (!ttisnil(res) ||  /* result is no nil? */
          (tm = fasttm(L, h->metatable, TM_INDEX)) == NULL) { /* or no TM? */
        setobj2s(L, val, res);
        return;
      }
```

否则，如果该对象不是一个Lua表，则查找该对象metatable表的__index字段，如果没有出错返回：

```c
else if (ttisnil(tm = luaT_gettmbyobj(L, t, TM_INDEX)))
      luaG_typeerror(L, t, "index");
```

如果前面两步都没有返回，则现在tm中存放的是查找到的metatable表的__index字段，如果是函数则进入该函数进行进一步的查找，否则则以tm来回到第一步进行进一步的查找:

```c
 if (ttisfunction(tm)) {
      callTMres(L, val, tm, t, key);
      return;
    }
    t = tm;  /* else repeat with `tm' */
```

如果对上面这个步骤不太理解，以最开始的实现继承的Lua代码来解释。
只看关键的几句代码。
1.  在base.lua的new中，设置metatable表的__index字段。前面的章节提到，module函数调用之后，会在__G表中新增一个表，对应的key就是”base”，也就是__G["base"] = {}，而这个表中存放的数据，就是该模块的全局函数和全局变量。因此，” setmetatable( obj, { __index = base } )”的作用就是，obj这个表，对它进行查找成员操作的时候，如果在这个表中查找不到则进入这个__index表来进行进一步的查找，也就是根据这样就可以查找到base模块中的全局函数和全局变量。
```lua
    module( "base", package.seeall )
    .....
    setmetatable( obj, { __index = base } )
```
2.   在test.lua文件中，其他部分与base一致，也是将该对象的metatable的__index赋值为obj。而最开始的“setmetatable( test, {__index = base} )”，则是将test表的metatable的__index赋值为base。

结合两部分的讲解，如果某成员在obj中查找不到，就会继续进入base表中进行查找。这是一个层次递进的关系。但是另一方面，效率上会有一定的影响。这并不是Lua中实现继承的唯一方式，云风在他的代码中也实现了Lua的继承，但是他的做法是将父类的成员直接赋值给子类。
