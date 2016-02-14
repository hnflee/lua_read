# Lua5.1.4代码分析(十一)-函数的调用

上一篇中,已经提到了函数的信息是如何在lua中保存的,本篇继续来讲解函数的调用机制.

首先谈谈函数的参数是如何处理的.
在分析函数的定义时,首先调用parlist处理函数的参数.简单起见,这里考虑函数的参数是确定的情况:

```c
static void parlist (LexState *ls) {
  /* parlist -> [ param { `,' param } ] */
  FuncState *fs = ls->fs;
  Proto *f = fs->f;
  int nparams = 0;
  f->is_vararg = 0;
  if (ls->t.token != ')') {  /* is `parlist' not empty? */
    do {
      switch (ls->t.token) {
        case TK_NAME: {  /* param -> NAME */
          new_localvar(ls, str_checkname(ls), nparams++);
          break;
        }
```

在这里,如果函数的参数是一个ID(TK_NAME)的情况,则调用new_localvar函数为这个变量预留一个空间保存这个变量,这样在函数将来被调用时,该参数实际是做为函数体的局部变量存在的.

于是,这样就基本完成了这个函数的解析.简单回忆一下:
1.  函数的内容是存放在结构体FuncState中的,函数内部如果再有函数,则依次存放在FuncState结构体的Proto *f数组中.
2.  对于一个lua源文件而言,即使内部没有定义任何函数,也会有一个默认的FuncState结构体,这一点在后面讲到Lua模块时再展开.
3.  函数做为lua中的first class,实际上也是一种类型的数据,于是存放和查找这个函数数据的过程,跟一般的查找其他类型的局部,全局等其他类型的数据一样样的.
4.  函数的参数是在分析函数体的时候确定下来的,实际上做为函数内的局部变量存在.

下面来看看如何调用一个函数.

调用一个函数的具体过程,实际上走的是如下流程:

```c
static void primaryexp (LexState *ls, expdesc *v) {
  /* primaryexp ->
        prefixexp { `.' NAME | `[' exp `]' | `:' NAME funcargs | funcargs } */
  FuncState *fs = ls->fs;
  prefixexp(ls, v);
  //.....
    for (;;) {
    switch (ls->t.token) {
          case '(': case TK_STRING: case '{': {  /* funcargs */
        luaK_exp2nextreg(fs, v);
        funcargs(ls, v);
        break;
      }
      default: return;
    }
  }
}
```

简单解释一下这个流程:
1.  首先调用prefixexp函数去解析将要调用的函数名,解析的结果存放在v中.
2.  调用luaK_exp2nextreg解析1)中解析出来的v到寄存器中,就我们前面的讨论结果,函数名可能是全局变量(GET_GLOBAL处理),也可能是局部变量(MOVE指令处理).
3.  调用funcargs进行调用函数的准备.因此这里的重点是第3)步,下面来详细看看.

还是考虑最简单的情况,没有可变参数的情况.那么传入函数的参数就是以”,”进行分割的N个变量,于是调用explist1函数依次将这些参数解析出来.前面已经提到,在分析函数的时候,实际上已经为这些参数保留了空间,将它们做为函数的局部变量处理.那么,大体上就应该是这样的一个流程:

```lua
function test(a, b)
   -- 这里暂且忽略函数体
end
上面这个函数定义相当于:
function test()
   -- 这里预留函数寄存器的两个位置保存a,b参数,把它们做为函数的局部变量处理,但是此时没有值
   local a = nil
   local b = nil
   -- 这里暂且忽略函数体
end

而当调用test(1,2)时,实际上函数体变为:
function test()
   local a = 1
   local b = 2
   -- 这里暂且忽略函数体
end
```

上面的奥秘在调用explist1中,这个函数依次解析格式为: “param [, param]“的函数参数列表,解析了一个参数就把一个值赋值到对应函数寄存器中,这样就把函数的形参和实参值对应上了.

继续往下看funcargs的代码.
首先来看这一句代码:

```c
base = f->u.s.info;  /* base register for call */
```

这里的f是前面解析成功的函数名,因此它的信息存放的是解析成功(也就是调用GET_GLOBAL或者MOVE之后赋值)的寄存器地址.因此base存放的是函数名的位置.

紧跟着:

```c
nparams = fs->freereg - (base+1);
```
得到函数参数的数量,因为函数的参数已经被逐个解析到了函数的寄存器中,这样两者的差值就是调用函数的参数数量.

最后初始化OPCODE表示调用函数:

```c
init_exp(f, VCALL, luaK_codeABC(fs, OP_CALL, base, nparams+1, 2));
```
这里的几个参数中,A(也就是上面的base)表示的是函数体信息当前存放到哪个寄存器位置,参数B(也就是nparams + 1)表示的是函数参数数量 + 1,参数C(也就是2)存放的是函数的返回值,为什么这里写死为2,后续再做解释.

以上完成了函数调用准备阶段.来看看虚拟机是如何执行相关的指令的.

```c
 case OP_CALL: {
        int b = GETARG_B(i);
        int nresults = GETARG_C(i) - 1;
        if (b != 0) L->top = ra+b;  /* else previous instruction set top */
        L->savedpc = pc;
        switch (luaD_precall(L, ra, nresults)) {
          case PCRLUA: {
            nexeccalls++;
            goto reentry;  /* restart luaV_execute over new Lua function */
          }
          case PCRC: {
            /* it was a C function (`precall' called it); adjust results */
            if (nresults >= 0) L->top = L->ci->top;
            base = L->base;
            continue;
          }
          default: {
            return;  /* yield */
          }
        }
      }
```

虚拟机调用函数的代码除了这部分之外,还要进入函数luaD_precall中分析一下,由于代码比较多,不具体详述.但是需要特别注意的是Lua栈的变化,我把涉及到的几句代码列出:
```c
lvm.c:585
if (b != 0) L->top = ra+b;  /* else previous instruction set top */

函数luaD_precall中:
    if (!p->is_vararg) {  /* no varargs? */
      base = func + 1;
      if (L->top > base + p->numparams)
        L->top = base + p->numparams;
    }       

L->base = ci->base = base;
```
以一个例子为例讲解这个过程Lua栈发生的变化:
前述中的ra指向的是解析函数完毕之后的数据,在调用函数之前,L->top = ra + 1,而如果这个函数是三个参数的函数,那么最终L->top = ra + 4,因为在ra之后紧跟着的三个位置是存放传入函数的参数.
而从”base = func + 1;”可知,base = ra + 1.同时，从语句”base = func + 1″可以看出，base始终在func的下一个栈位置。

因此调用之后的函数栈,它的base = ra + 1,而在base和top之间空出的几个位置,是用来存放传入函数的参数.而函数返回时,自然是将这个过程反过来做一遍.

以上过程,涉及到lua栈的不少概念.我可能表述的还不够清楚,但是暂时也只能这样了.有疑问的,可以自己跟着分析和代码一起看看:)

再重复一下,这里仅仅是最简单的情况,还有需要复杂的情况没有考虑:可变参数,tailcall(把一个函数的调用做为另一个函数的返回值),函数的upvalue等等,都还没有讲解.以后有机会再展开吧.