# Lua5.1.4代码分析(八)-逻辑跳转类操作

我原来以为逻辑跳转类指令是很简单的,实际上花了我很多时间才看的比较明白.未来也许还需要重新回顾这部分代码.

暂时不展开复杂的讨论,所有的逻辑跳转类指令最后都无非是这样的形式:

```
如果条件成立跳转到lable1,否则跳转到lable2:

label1:
    条件成立的处理,跳转到出口

label2:
     条件不成立的处理,跳转到出口

出口:
    处理收尾工作
```

比如,以下的一段C代码:

```C
if (cond)
   func1();
else
   func2();
func_end();
```

可以套用上面的模式改写翻译为:

```C
if (cond)
   goto label1;
goto lable2;

label1:
   func1();
   goto label_end;

label2:
   func2();
   goto label_end;

lable_end:
   func_end();
```

虽然都可以套用这个模式来将条件语句翻译为带有跳转指令的代码,但是有以下几点需要注意:
1. 有一些跳转语句实际上不必要的,比如在label2中的”goto label_end;”这一句实际上不必要的,因为紧跟着的语句就是label_end.
2. 这是这里最关键的问题,跳转位置实际上在生成跳转语句的时候经常是不知道的.比如最开始判断cond是否成立来决定跳转位置的时候,实际上label1,label2还并未生成.


对于第一个问题,可以改写代码,从而删去一些多余的语句,改写如下:

```C
if (cond)
   goto lable1;

label2:
   func2();
   goto label_end;

label1:
   func1();

lable_end:
   func_end();
```

对于第二个问题,在编译原理的理论中,使用一种称为”回填”(backpatch)的技术来进行处理.它的做法是,生成跳转语句时,将当前还不知道位置,但是都要跳转到同一个位置的语句链接在一起,形成一个空悬跳转语句的链表,在后面找到跳转位置时,再将跳转位置遍历之前的链表填充回去.由于跳转无非就是条件为真和为假两种情况的跳转,所以同一个表达式只有两个跳转链表,一般称为truelist和falselist.

还是以开始的例子来解释这个过程.

```C
if (cond)
        // 生成一个跳转语句,此时label1位置未知,因此生成跳转语句的跳转点加入cond的truelist

label2:
   func2();
   // 生成一个跳转语句,此时label_end位置未知,因此生成跳转语句的跳转点加入cond的falselist

label1:
   func1();

lable_end:
   func_end();
```

这里只是最简单的情况,如果有多个elseif的情况处理,那么truelist和falselise就可能不止只有一个元素.

从这里看出,回填技术涉及到两个操作:
1.  将当前生成的跳转语句加入到某个空悬链表中
2.  以某个位置的数据,回填1中生成的空悬链表的悬空地址.

可以把空悬链表看做是这样的链表:它将一系列空悬的跳转点链接在一起,而它们都将跳转到同一个位置,而当这个位置已知的时候,再将这个地址回填到这些空悬跳转点上完成跳转位置的修正.

类似的,可以将上面的例子继续扩宽为”if con1 elseif cond2 …. else if condn else “的格式,具体不再详述.可以参考龙书中关于中间代码生成部分的讲解.

有了上面的理论基础,可以来看看Lua中相关的代码.

```C
static void ifstat (LexState *ls, int line) {
  FuncState *fs = ls->fs;
  int flist;
  int escapelist = NO_JUMP;
  flist = test_then_block(ls);  /* IF cond THEN block */
  while (ls->t.token == TK_ELSEIF) {
    luaK_concat(fs, &escapelist, luaK_jump(fs));
    luaK_patchtohere(fs, flist);
    flist = test_then_block(ls);  /* ELSEIF cond THEN block */
  }
  if (ls->t.token == TK_ELSE) {
    luaK_concat(fs, &escapelist, luaK_jump(fs));
    // 使用返回的flist修正悬空链表中的第一个悬空结点
    luaK_patchtohere(fs, flist);
    luaX_next(ls);  /* skip ELSE (after patch, for correct line info) */
    block(ls);  /* `else' part */
  }
  else
    luaK_concat(fs, &escapelist, flist);
  // 使用escapelist链表来修正jpc
  luaK_patchtohere(fs, escapelist);
  check_match(ls, TK_END, TK_IF, line);
}
```
我们同样以最简单的Lua代码来讲解:
```C
if x then
   x = 1
else
   x = 2
end
x = 3
```

代码:

```C
flist = test_then_block(ls);  /* IF cond THEN block */
```

是用于读取第一个条件,并且读入第一个block的代码,对应于上面列举出的Lua代码,就是:

```C
if x then
   x = 1
```

可以看到,判断了x之后,紧跟着就是x为true时的情况处理,也就是x = 1.而为false的情况是悬空的.
因此整个语句”flist = test_then_block(ls);”的作用是:测试并且读取block,而true跳转到block语句的开头,而false情况是悬空的.

接着往下看,由于没有elseif的情况处理,直接看else的情况处理,对应的代码是:

```C
 if (ls->t.token == TK_ELSE) {
    luaK_concat(fs, &escapelist, luaK_jump(fs));
    luaK_patchtohere(fs, flist);
    luaX_next(ls);  /* skip ELSE (after patch, for correct line info) */
    block(ls);  /* `else' part */
  }
```

首先调用luaK_concat(fs, &escapelist, luaK_jump(fs));生成一个空悬的跳转语句,并且将它加入到escapelist中.
其次调用luaK_patchtohere(fs, flist);回填前面返回的空悬flist,也就是说,前面的flist跳转地址为下一条生成的指令,也就是紧跟着的block.

最后走出了条件语句,再调用luaK_patchtohere(fs, escapelist);函数将之前的空悬列表回填为下一条指令(也就是最后的x = 3执行之前).

这样,对于这段Lua代码,所有的跳转位置都已经生成了.

Lua源码中涉及到这部分的几个关键函数和数据结构分别是:
1.  结构体struct expdesc中的成员t,f,顾名思义,它们就是存放这个表达式的truelist和falselist.但是需要注意的是,这两个成员都是int类型,而不是一个显示的链表结构体.原因是它们存放的都是Lua opcode,可以用里面的偏移量来保存链表成员.这也是Lua代码中实现的很精巧的部分.Lua源码中提供函数getjump根据传入的int值来遍历整个空悬链表的.

2.  luaK_concat函数:将某一个指令l2加入到l1存放的空悬链表中.这表示,它们未来的跳转位置都是一样的,当知道跳转位置之后,就遍历这个链表回填位置.

3.  luaK_patchtohere函数:将空悬跳转点加入jpc维持的空悬链表中.

那么何时真正的进行回填呢?
每次生成一个新的opcode,最终都会走到函数luaK_code中,它会调用dischargejpc函数,其中会判断如果当前的jpc如果不是无效的跳转位置(NO_JUMP),那么就会遍历jpc维持的空悬链表,以当前pc的数据遍历这个链表的空悬跳转点回填跳转点.

换句话说,如果你想要跳转到下一个指令的位置,只需要调用luaK_patchtohere函数即可,在生成下一条指令之前,自动会将下一条指令的位置写入.

写了这么多,不知道写清楚没有,有兴趣的可以结合着龙书第一版第八章看这部分代码.

