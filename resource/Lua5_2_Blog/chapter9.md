# 编译系统(1) 概述

Lua是一个轻量级高效率的语言。这种轻量级和高效率不仅体现在它本身虚拟机的运行效率上，而且也体现在他整个的编译系统的实现上。因为绝大多数的lua脚本需要运行期动态的加载编译，如果编译过程本身非常耗时，或者占用很多的内存，也同样会影响到整体的运行效率，使你感觉这个语言不够“动态”。正是因为编译系统实现的非常出色，我们在实际使用lua时基本感觉不到这个过程的存在。

要实现一个Lua的编译系统可能不是很困难，但要高效的实现，还是有一定挑战的。这需要一些精妙的设计和实现技巧，也值得我们花一些时间去一探究竟。


### **输入与输出**

编译系统的工作就是将符合语法规则的chunk转换成可运行的closure。要了解编译系统，首先要了解作为输入的chunk和最为输出的closure以及他们的对应关系。

说到closure，就不能不先说一下proto。closure对象是lua运行期一个函数的实例对象，我们在运行期调用的都是一个closure。而proto对象是lua内部代表一个closure原型的对象，有关函数的大部分信息都保存在这里。这些信息包括：

* 指令列表：包含了函数编译后生成的虚拟机指令。
* 常量表：这个函数运行期需要的所有常量，在指令中，常量使用常量表id进行索引。
* 子proto表：所有内嵌于这个函数的proto列表，在OP_CLOSURE指令中的proto id就是索引的这个表。
* 局部变量描述：这个函数使用到的所有局部变量名称，以及生命期。由于所有的局部变量运行期都被转化成了寄存器id，所以这些信息只是debug使用。
* Upvalue描述：设个函数所使用到的Upvalue的描述，用来在创建closure时初始化Upvalue。

每个closure都对应着自己的proto，而运行期一个proto可以产生多个closure来代表这个函数实例。

![](https://git.gitbook.com/raw/wyyhzc/gifs/master/1358486658_7085.png?token=d3l5aHpjOjkyODA1MGVkLTMxZDEtNDFmOS04MjY3LWU1YzdmNjU4M2U3Nw%3D%3D)

由此可见，closure是运行期的对象，与运行期关系更大；而与编译期相关的其实是proto对象，他才是编译过程真正需要生成的目标对象。

Chunk代表一段符合Lua的语法的代码。我们可以调用lua_load api，将一个chunk进行编译。lua_load根据当前chunk生成一个mainfunc proto，然后为这个proto创建一个closure放到当前的栈顶，等待接下来的执行。Chunk内部的每个function statement也都会生成一个对应的proto，保存在外层函数的子函数列表中。所有最外层的function statement的proto会被保存到mainfunc proto的子函数列表中。所以，整个编译过程会生成一个以mainfunc为根节点的proto树。

![](https://git.gitbook.com/raw/wyyhzc/gifs/master/1358488840_6386.png?token=d3l5aHpjOjkyODA1MGVkLTMxZDEtNDFmOS04MjY3LWU1YzdmNjU4M2U3Nw%3D%3D)

###**简介**
 



按照功能划分，整个编译系统被划分成以下3个模块：

* 词法分析模块llex.h .c
* 语法分析模块lparser.h .c
* 指令生成模块lcode.h .c

Lua并没有使用llex和yacc，而是使用完全手写的词法和语法分析器。使用手写分析器的原因首先是考虑到效率。并且yacc/bison本身生成的代码在可移植性上有一些问题，无发达到Lua高可移植性的设计目标。还有就是手写分析器可以在C stack上面分配编译过程中需要的数据对象，这个我们后面会讲到。对于一个chunk，Lua在对其分析的过程中直接生成最终的指令，没有多余的对源代码或语法结构的遍历。也就是说Lua对源代码进行一次遍历就生成最终结果。


### **词法分析**

Lua的词法分析模块比较简单而且独立，就是将源代码拆分成一个个token，提供给语法分析使用。语法分析程序会调用luaX_next来获取下一个单词，然后进行语法分析。

```
    typedef union {  
      lua_Number r;  
      TString *ts;  
    } SemInfo;  /* semantics information */  
      
      
    typedef struct Token {  
      int token;  
      SemInfo seminfo;  
    } Token;  
```

Token用来表示一个单词，它包括类型token和语义seminfo。类型使用一个int来表示，既可以是一个字符，也可以是一个enum RESERVED。enum RESERVED从257开始，就是为了留给字符使用。如果类型是一个TK_NUMBER，seminfo.r就用来表示这个数字；如果是TK_NAME或者TK_STRING，seminfo.ts就表示对应的字符串。

LexState结构体的用途其实与其名称不是很贴切。LexState不仅用于保存当前的词法分析状态信息，而且也保存了整个编译系统的全局状态，这个我们在后面会讲到。


### **语法分析和指令生成**

语法分析器是整个编译过程的驱动器。通过对luaY_parser函数的调用，启动整个编译过程。语法分析采用“递归下降”的方法，从词法分析器中读取下一个token，然后根据这个token和lua的语法规则，将高层的语法规则分解成底层的语法规则，进一步进行分析。根据语法，“递归”在lua语法中只出现在两个地方，一个是statement函数，一个是subexpr函数。这也就是在这两个函数中调用enterlevel和leaveleavel，对当前调用深度进行检测的原因。如果递归太深，编译会报错。在分析的过程中，词法分析器会调用指令生成器，直接生成最终的指令。

从宏观上讲，整个编译过程就是生成proto tree的过程。语法分析器从mainfunc出发，开始分析和生成mainfunc的proto。在生成一个proto的过程中，生成的指令直接保存到proto的指令列表中。当遇到function statement或者local function statement时，首先生成子函数的proto，然后回来继续。通过这样的遍历方式，最终构建出一个proto tree。

在编译过程中，使用FuncState结构体来保存一个函数编译的状态数据。每个FuncState都有一个prev变量用来引用外围函数的FuncState，使当前所有没有分析完成的FuncState形成一个栈结构。栈底是mainfunc的FuncState，栈顶是当前正在分析的FuncState。每当开始分析一个新的函数时，会创建一个新的FuncState与之对应，将当前的FuncState保存在新的FuncState的prev中，并将当前的FuncState指向新的FuncState，这相当于压栈(open_func)。等待这个新函数分析完成后，当前的FuncState就没用了，将当前的FuncState恢复成原来的FuncState，这相当于弹栈(close_func)。

Dyndata是一个全局数据，他本身也是一个栈。对应上面的FuncState栈，Dyndata保存了每个FuncState对应的局部变量描述列表，goto列表和label列表。这些数据会跟着当前FuncState进行压栈和弹栈。


前面说过，整个编译系统的全局状态都保存在LexState中。LexState中与全局状态相关的主要是两个变量。fs指向当前正在编译的函数的FuncState。而dyd则指向全局的Dyndata数据。


![](https://git.gitbook.com/raw/wyyhzc/gifs/master/1358500685_9493.png?token=d3l5aHpjOjkyODA1MGVkLTMxZDEtNDFmOS04MjY3LWU1YzdmNjU4M2U3Nw%3D%3D)

FuncState本身通过f保存对于Proto的引用。所有过程中生成的指令都直接保存到Proto的指令列表中。FuncState还通过h引用到一个table，用于在编译过程中生成proto的常量表。这个table使用常量值作为key，常量的id作为value。当编译过程中发现一个常量时，会首先使用这个常量值在这个表中查找。如果找到，说明前面已经将这个常量加入到常量表了，直接使用其id。否者，向常量表中添加一个常量，并相应的添加到这个表中。

对于一个FuncState本身的分析也是有层次关系的。一个函数本身使用block来控制局部变量的有效范围。函数本身就是一个block，里面的想while statement，for statement等等，还会形成子block。函数内的这些block会形成一个以函数本身block为根节点的block tree。Lua使用BlockCnt来保存一个block的数据。与FuncState的分析方法类似，BlockCnt使用一个previous变量保存外围block的引用，形成一个栈结构。enterblock和leaveblock函数负责压栈和弹栈。在FuncState中，bl用来指向当前的block。

![](https://git.gitbook.com/raw/wyyhzc/gifs/master/1358503549_4311.png?token=d3l5aHpjOjkyODA1MGVkLTMxZDEtNDFmOS04MjY3LWU1YzdmNjU4M2U3Nw%3D%3D)

纵观整个语法分析过程，Lua其实就是按照深度优先的顺序，遍历了FuncState tree，以及子结构BlockCnt tree。在遍历的过程中，所有的编译状态(FuncState,BlockCnt)都是在C stack中保存，这就表示只保存当前正在处理的编译状态，而那些已经处理完成的在弹栈时被丢弃。在分析具体的语法结构时也是如此，Lua并被有完整的构建一个语法树对象，而是将过程中的语法结构保存在函数栈中，分析完立刻丢弃。所以就算用Lua来分析一个很大的程序，也不会占用过多的内存。不过，这也为指令生成带来了很多麻烦。比如向后跳转，在分析过程中还无法知道具体的跳转地址。如果构建了完整的语法树，就可以在最后去解决这些未决的地址。Lua使用了一些技巧来解决这些问题，我会在后面的文章中详细讲述。

在C stack中保存编译状态数据还有一个原因，是和编译系统的报错机制相关。编译系统整体的报错机制采用与虚拟机运行期一致的异常处理机制，也就是longjump。当出错时，直接跳出到最外层进行处理，此时所有当前的编译状态数据要能自动销毁掉。