# Lua5.1.4代码分析(二十一)-表的创建和初始化

在讲解之前，先来简单回顾一下Lua表的初始化语法。

在Lua中，表是唯一的数据结构，可以使用它，模拟hash表，数组，链表，树等一切常用的数据结构。Lua表分为数组部分和hash部分。比如：


```lua
local t = {1,2,3,4,5}
```

以上分配一个Lua数组，依次为1到5.

而如果要初始化hash部分，则需要指定key，有两种方式：

```lua
local t = {a="test"}
local t = {["a"]="test"}
```
以上都指定了key为”a”的元素对应的值是”test”（注意一些上面两种情况key分别可以加引号和不加引号的）。

OK，现在可以来看Lua表创建相关的操作。涉及到这部分的，是两个OPCODE：
1.  NEWTABLE指令。指令域A指定的是所要创建的表在Lua栈中的地址，而B,C则分别指定的是创建表时数组和hash部分的初始大小。

2.  SETLIST指令。需要特别说明的是，这个指令仅能用于初始化Lua表的数组部分时使用，hash部分没有作用。指令域A同样指定的是所要初始化的表在Lua栈中的地址，B指定的是初始化时数组的数量，而C指定的是BLOCK的数量。这里需要做一个说明。在Lua中有一个特殊的常量，叫FPF（fields per flush），可以简单的理解为，每次调用SETLIST指令时，写入数组的数量最多可以有多少，Lua中这个常量定义为50.于是，假如这里要初始化一个有60个元素的数组，那么将会拆分成两个SETLIST指令，第一个SETLIST指令，B为50，C为1，而第二个SETLIST指令，B为10而C为0.
实际上，SETLIST指令还是有点复杂的。需要再继续了解一下几个知识点。
首先，lopcodes.h中对这个指令的注释为：
```c
OP_SETLIST,/*   A B C   R(A)[(C-1)*FPF+i] := R(A+i), 1 <= i <= B        */
```
需要注意的是，A在这里既指定了表的栈位置，还有另一层含义从"="右边可知，A在栈中紧跟着的数据是需要初始化给A数组的数据，所以A在这个指令中负担了两个数据的指示。换言之，当在A位置创建了这个Lua表之后，紧跟着这个Lua表的数据（数量由B指定）则是准备初始化给Lua表的数据。
3.  Lua还要处理某些情况下，数组元素可变的情况，比如:
```lua
local t = {func()}
```
可以看到，此时数组元素的数量是不确定的，依赖于函数的返回值，而当解析到这个点时，也并不知道func的具体情况。Lua在这里的处理是将B置为0，表示从A+1位置开始直到这个函数栈的栈顶位置之间的元素全部用来初始化这个Lua表的数组部分。
4.  C也有可能为0，但是这种情况很少有，仅当初始化数组的数量非常大的时候出现，这里就不做分析了（因为要模拟这种情况有些蛋疼）。
有了前面的储备，可以来看看Lua源码中相关的实现。


分析Lua表创建部分的入口函数是lparser.c中的constructor函数。
首先，函数调用pc = luaK_codeABC(fs, OP_NEWTABLE, 0, 0, 0);生成一个NEWTABLE指令，注意在这里，B/C部分都是0，从前面的分析知道，这两部分分别指定的是Lua表的数组和hash部分的初始尺寸，因为在这里这两部分的大小并不知道，所以先填0，而保存在pc中是要保存这个生成的NEWTABLE指令，后面需要对B/C部分进行改写，填充数组和hash部分的尺寸。

紧跟着，在解析Lua表初始化的整个流程中，使用了结构体ConsControl：

```c
struct ConsControl {
  expdesc v;  /* last list item read */
  expdesc *t;  /* table descriptor */
  int nh;  /* total number of `record' elements */
  int na;  /* total number of array elements */
  int tostore;  /* number of array elements pending to be stored */
};

```

每一项的含义分别是，v表示的是上一个解析到表元素，它可能是一个key-value形式的赋值（此时是初始化一个hash元素），也有可能是单独的元素（此时是初始化一个数组元素）；t是一个指针，存放的是待初始化的Lua表；nh和na分别表示表的hash和数组部分尺寸，解析过程中将用这两个变量记录以便在最后重新填充前面的NEWTABLE的B/C部分；tostore则是存放的当前已经有多少数组元素待存放到Lua表中，当这个值达到FPF时，根据上面的分析则生成一个SETLIST指令，然后重新值0进入下一个元素的处理。


```c
 509   checknext(ls, '{');
 510   do {
 511     lua_assert(cc.v.k == VVOID || cc.tostore > 0);
 512     if (ls->t.token == '}') break;
 513     closelistfield(fs, &cc);
 514     switch(ls->t.token) {
 515       case TK_NAME: {  /* may be listfields or recfields */
 516         luaX_lookahead(ls);
 517         if (ls->lookahead.token != '=')  /* expression? */
 518           listfield(ls, &cc);
 519         else
 520           recfield(ls, &cc);
 521         break;
 522       }
 523       case '[': {  /* constructor_item -> recfield */
 524         recfield(ls, &cc);
 525         break;
 526       }
 527       default: {  /* constructor_part -> listfield */
 528         listfield(ls, &cc);
 529         break;
 530       }
 531     }
 532   } while (testnext(ls, ',') || testnext(ls, ';'));
 533   check_match(ls, '}', '{', line);
 534   lastlistfield(fs, &cc);
 535   SETARG_B(fs->f->code[pc], luaO_int2fb(cc.na)); /* set initial array size */
 536   SETARG_C(fs->f->code[pc], luaO_int2fb(cc.nh));  /* set initial table size */
 ```
 
 这个分析过程的主体部分，是一个循环，循环的终止条件是遇到了"}"符号，则该数组的初始化部分完成。
每次循环做以下的事情：
1.  调用closelistfield函数。它是对数组元素做处理。首先将上一个分析到的数组元素，写入到当前的Lua栈中，这一点可以结合前面分析SETLIST指令来看。同时，如果当前的tostore数量达到FPF时，则生成SETLIST指令，这一点前面也做了分析。

2.  后就是两种情况的处理：hash和数组部分，可以参看最开始Lua表初始化的语法就能知道什么语法是用于初始化hash部分，什么语法是初始化数组部分的了。分别调用的是recfield和listfield函数。
listfield函数相对简单，需要判断当前表的数组元素是不是超过了限制，同时增加na和tostore计数。
recfield稍微复杂一点，还涉及到另一个指令SETTABLE，暂时跳过下一节再解释，现在知道它肯定会增加na计数就可以了。

3.  最后，由于初始化Lua表时，不同的元素之间是以","或者";"做分割的，所以在遇到"}"退出循环之后，还有最后一个元素没有处理，于是还要调用lastlistfield函数进行处理。
lastlistfield函数要处理的情况，就是前面分析过的，初始化过程中是不是遇到了函数返回值的情况，如果有则生成的SETLIST指令的域B要为0.

4.  最后就是根据分析过程中得到的na，nh数量重新填充NEWTABLE指令的B/C域了。