# Lua5.1.4代码分析(二十)-函数的返回参数

Lua中，函数的返回参数数量可能会随着赋值表达式左边的情况而进行调整。

比如，同样的函数f()，本来返回两个参数，而如果是表达式A = f()，则第二个返回参数将被抛弃。同样的，如果是表达式A,B,C = f()，则C则被复制为nil。这又涉及到Lua的堆栈调整了。

首先来看函数的返回数量由什么来决定的。它是由OP_CALL这个指令的第三个参数决定的。在Lua虚拟机中，执行到OP_CALL指令时，得到这个参数存入变量nresults中，然后调用luaD_precall将其保存起来。而在调用函数结束之后，在函数luaD_poscall中对返回的堆栈进行调整：

```c
// 结束完一次函数调用(无论是C还是lua函数)的处理, firstResult是函数第一个返回值的地址
int luaD_poscall (lua_State *L, StkId firstResult) {
  StkId res;
  int wanted, i;
  CallInfo *ci;
  if (L->hookmask & LUA_MASKRET)
    firstResult = callrethooks(L, firstResult);
  // 得到当时的CallInfo指针
  ci = L->ci--;
  res = ci->func;  /* res == final position of 1st result */
  // 本来需要有多少返回值
  wanted = ci->nresults;
  // 把base和savepc指针置回调用前的位置
  L->base = (ci - 1)->base;  /* restore base */
  L->savedpc = (ci - 1)->savedpc;  /* restore savedpc */
  /* move results to correct place */
  // 返回值压入栈中
  for (i = wanted; i != 0 && firstResult < L->top; i--)
    setobjs2s(L, res++, firstResult++);
  // 剩余的返回值置nil
  while (i-- > 0)
    setnilvalue(res++);
  // 可以将top指针置回调用之前的位置了
  L->top = res;
  return (wanted - LUA_MULTRET);  /* 0 if wanted == LUA_MULTRET */
}
``` 

这里的wanted表示的表达式左边需要返回多少，而firstResult到L->top之间的元素则是函数内实际返回的，如果wanted需要的元素数量比这个多，将在后面补零。注意返回的数量有OP_RETURN指令决定。

由以上可以看出，这里有两个指令起作用。首先是OP_CALL指令，它表示表达式需要返回几个参数；其次OP_RETURN指令，它表示函数内部实际返回的参数数量。

如此，分别来看看这两部分指令的生成。

首先来看OP_CALL指令。对于赋值表达式，在分析的时候是调用函数localstat,这里将会首先对表达式右边做分析，如果判断这是调用一个函数的话，最后将走入函数funcargs中，这个过程前面已经讲解过不再分析，在第一次生成这个指令的时候默认将参数C设置为2，而这个部分就是存放函数返回参数数量的：
init_exp(f, VCALL, luaK_codeABC(fs, OP_CALL, base, nparams+1, 2));

如果看到lopcodes.h中对OP_CALL指令的解释就可以知道，这个指令的C参数是返回参数数量+1:
OP_CALL,/* A B C R(A), … ,R(A+C-2) := R(A)(R(A+1), … ,R(A+B-1)) */

换言之，在首次生成OP_CALL指令的时候，默认返回1个参数。

回到localstat函数中，这里已经对表达式右边做了分析，接着将会分析表达式的左边，得到在“=”左边有多少个变量等着赋值：

```c
static void localstat (LexState *ls) {
  /* stat -> LOCAL NAME {`,' NAME} [`=' explist1] */
  int nvars = 0;
  int nexps;
  expdesc e;
  do {
    new_localvar(ls, str_checkname(ls), nvars++);
  } while (testnext(ls, ','));
  if (testnext(ls, '='))
    nexps = explist1(ls, &e);
  else {
    e.k = VVOID;
    nexps = 0;
  }
  adjust_assign(ls, nvars, nexps, &e);
  adjustlocalvars(ls, nvars);
}
```

可以看到，“=”号左边表达式的数量存入到变量nexps中，再调用函数adjust_assign中,它将判断如果表达式右边是一个函数调用的话，会使用传入的nexps参数调整函数返回值的数量，也就是OP_CALL中C参数的数值，这个过程在函数luaK_setreturns中：

```c
void luaK_setreturns (FuncState *fs, expdesc *e, int nresults) {
  if (e->k == VCALL) {  /* expression is an open function call? */
    SETARG_C(getcode(fs, e), nresults+1);
  }
  else if (e->k == VVARARG) {
    SETARG_B(getcode(fs, e), nresults+1);
    SETARG_A(getcode(fs, e), fs->freereg);
    luaK_reserveregs(fs, 1);
  }
}
```

以上，分析了OP_CALL指令调整C参数的过程，再来看看OP_RETURN指令的处理流程。

这个过程对比前面的OP_CALL就相对简单，它调用的是函数luaK_ret：

```c
void luaK_ret (FuncState *fs, int first, int nret) {
  luaK_codeABC(fs, OP_RETURN, first, nret+1, 0);
}
```

来看看这个函数两个被调用的地方。第一个地方是函数close_func，显然这是分析函数结束之后调用的一个函数。Lua中，无论一个函数是否返回值，都会在最后插入OP_RETURN指令，所以无论前面有没有显式的写return XXX，在close_func函数中都会补上这个指令，只不过写入的是luaK_ret(fs, 0, 0);，结合前面的流程可以看到，这表示函数没有任何返回值，所以luaD_poscall函数中wanted部分全部填nil。

再来看显式的写return XXX之后的处理，这调用的retstat函数。

```c
// 处理return指令
static void retstat (LexState *ls) {
  /* stat -> RETURN explist */
  FuncState *fs = ls->fs;
  expdesc e;
  int first, nret;  /* registers with returned values */
  // 略过return关键字，读入下一个token
  luaX_next(ls);  /* skip RETURN */
  // 如果紧跟着是END表示函数结束，或者是;，这两种情况都是没有返回值
  if (block_follow(ls->t.token) || ls->t.token == ';')
    first = nret = 0;  /* return no values */
  else {
  // 读入表达式列表，返回值是表达式列表的表达式数量
    nret = explist1(ls, &e);  /* optional return values */
    // 如果是函数调用或者可变参数
    if (hasmultret(e.k)) {
      luaK_setmultret(fs, &e);
      if (e.k == VCALL && nret == 1) {  /* tail call? */
        SET_OPCODE(getcode(fs,&e), OP_TAILCALL);
        lua_assert(GETARG_A(getcode(fs,&e)) == fs->nactvar);
      }
      first = fs->nactvar;
      nret = LUA_MULTRET;  /* return all values */
    }
    else {
      // 如果表达式列表中仅有一个表达式
      if (nret == 1)  /* only one single value? */
        first = luaK_exp2anyreg(fs, &e);
      else {
        luaK_exp2nextreg(fs, &e);  /* values must go to the `stack' */
        first = fs->nactvar;  /* return all `active' values */
        lua_assert(nret == fs->freereg - first);
      }
    }
  }
  // first是起始寄存器索引，nret是数量
  luaK_ret(fs, first, nret);
}
```
以上，分析完了设置函数返回参数数量以及相关的处理流程。


