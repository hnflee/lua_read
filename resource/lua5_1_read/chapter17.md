# Lua5.1.4代码分析(十七)-可变参数函数的实现

可变函数参数的实现相对简单。

首先来看看Lua中可变参数函数的使用方式。如果函数传入的参数是”…”,则表示这里的参数是可变参数，在使用时，它们最终会存储到一个名为arg的数组中。

如前面对函数的解析里面提到，在对一个函数的参数进行解析时，会走到parlist函数中，这里处理了变参参数的情况：


```C 
static void parlist (LexState *ls) {
  /* parlist -> [ param { `,' param } ] */
  FuncState *fs = ls->fs;
  Proto *f = fs->f;
  int nparams = 0;
  f->is_vararg = 0;
  if (ls->t.token != ')') {  /* is `parlist' not empty? */
    do { 
      switch (ls->t.token) {
        case TK_DOTS: {  /* param -> `...' */
          luaX_next(ls);
#if defined(LUA_COMPAT_VARARG)
          /* use `arg' as default name */
          new_localvarliteral(ls, "arg", nparams++);
          f->is_vararg = VARARG_HASARG | VARARG_NEEDSARG;
#endif
          f->is_vararg |= VARARG_ISVARARG;
          break;
        }
        default: luaX_syntaxerror(ls, " or " LUA_QL("...") " expected");
      }
    } while (!f->is_vararg && testnext(ls, ','));
  }
  // ....
}
```

可以看到，如果解析器检测到函数定义时，传入的参数是”…”，则会打上一个标记位，并且定义了一个名为”arg”的局部变量给函数，而在真正传入参数调用函数时，这个标记位起了作用。
在luaD_precall函数中，也就是调用函数的预处理函数，判断如果是一个变参函数，则会调用函数adjust_varargs 进行处理这些变参：

```c
static StkId adjust_varargs (lua_State *L, Proto *p, int actual) {
  int i;
  int nfixargs = p->numparams;
  Table *htab = NULL;
  StkId base, fixed;
  for (; actual < nfixargs; ++actual)
    setnilvalue(L->top++);
#if defined(LUA_COMPAT_VARARG)
  if (p->is_vararg & VARARG_NEEDSARG) { /* compat. with old-style vararg? */
    int nvar = actual - nfixargs;  /* number of extra arguments */
    lua_assert(p->is_vararg & VARARG_HASARG);
    luaC_checkGC(L);
    htab = luaH_new(L, nvar, 1);  /* create `arg' table */
    for (i=0; itop - nvar + i);
    /* store counter in field `n' */
    setnvalue(luaH_setstr(L, htab, luaS_newliteral(L, "n")), cast_num(nvar));
  }
#endif
  /* move fixed parameters to final position */
  fixed = L->top - actual;  /* first fixed argument */
  base = L->top;  /* final position of first argument */
  for (i=0; itop++, fixed+i);
    setnilvalue(fixed+i);
  }
  /* add `arg' parameter */
  if (htab) {
    sethvalue(L, L->top++, htab);
    lua_assert(iswhite(obj2gco(htab)));
  }
  return base;
}
```

在这个函数里，如果判断前面的变参参数标记位置位，说明是个变参函数，那么将后面的参数以此放入名为arg的数组中，这样在具体函数内使用该arg数组时就可以取到传入的参数了。