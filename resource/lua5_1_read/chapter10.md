# Lua5.1.4代码分析(十)-函数的定义

Lua源码中,专门有一个结构体FuncState用来保存函数相关的信息.其实,即使没有创建任何函数,对于Lua而言也有一个最外层的FuncState数据.来看看这个结构体的定义:

```C
typedef struct FuncState {
  Proto *f;  /* current function header */
  Table *h;  /* table to find (and reuse) elements in  */
  struct FuncState *prev;  /* enclosing function */
  struct LexState *ls;  /* lexical state */
  struct lua_State *L;  /* copy of the Lua state */
  struct BlockCnt *bl;  /* chain of current blocks */
  int pc;  /* next position to code (equivalent to `ncode') */
  int lasttarget;   /* `pc' of last `jump target'  */
  int jpc;  /* list of pending jumps to `pc' */
  int freereg;  /* first free register */
  int nk;  /* number of elements in `k'  */
  int np;  /* number of elements in `p'  */
  short nlocvars;  /* number of elements in `locvars' */
  lu_byte nactvar;  /* number of active local variables  */
  upvaldesc upvalues[LUAI_MAXUPVALUES];  /* upvalues */
  unsigned short actvar[LUAI_MAXVARS];  /* declared-variable stack */
} FuncState;
```

其中的Proto结构体数组用于保存函数原型信息,包括函数体代码(opcode),之所以使用数组,是因为在某个函数内,可能存在多个局部函数.而prev指针就是指向这个函数的”父函数体”的指针.

比如以下代码:

```lua
function fun()
   function test()
   end
end
```

那么,在保存test函数原型的Proto数据就存放在保存fun函数的FuncState结构体的p数组中,反之,保存test函数的FuncState.prev指针就指向保存func函数的FuncState指针.

接着看Funcstate结构体的成员,actvar数组用于保存局部变量,比如函数的参数就是保存在这里.另外还有一个存放upval值的upvalues数组.这里有两种不同的处理.如果这个upval是父函数内的局部变量,则生成的是MOVE指令用于赋值;如果对于父函数而言也是它的upval,则生成GET_UPVAL指令用于赋值.

当开始处理一个函数的定义时,首先调用open_func函数,创建一个新的Proto结构体用于保存函数原型信息,接着将该函数的FuncState的prev指针指向父函数.
最后当函数处理完毕时,调用pushclosure函数将这个新的函数的信息push到父函数的Proto数组中.

最后,由于函数在Lua中是所谓的”first class type”,所以其实以下两段Lua代码是等价的:

```lua
function test()
end

test = function ()
end

```
也就是说,其实是生成一段代码,用于保存函数test的相关信息,之后再将这些信息赋值给变量test,这里的test可以是local,也可以是global的,这一点跟一般的变量无异.

所以在与函数定义相关的词法分析代码中:

```c
static void funcstat (LexState *ls, int line) {
  /* funcstat -> FUNCTION funcname body */
  int needself;
  expdesc v, b;
  luaX_next(ls);  /* skip FUNCTION */
  needself = funcname(ls, &v);
  body(ls, &b, needself, line);
  luaK_storevar(ls->fs, &v, &b);
  luaK_fixline(ls->fs, line);  /* definition `happens' in the first line */
}
```


上面的变量v首先在funcname函数中获得该函数的函数名,变量b在进入函数body之后可以得到函数体相关的内容.在这之后的luaK_storevar调用,就是把b的值赋值给v,也就是前面提到的函数体赋值给函数名.



