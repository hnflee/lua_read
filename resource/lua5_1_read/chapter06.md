# Lua5.1.4代码分析(六)Opcode格式

了解了Lua虚拟机和栈的结构之后,在正式进入分析各种Lua的操作之前,还需要简单了解Lua Opcode的格式.

Lua的opcode格式分为三类,在lopcode.h中有定义:
enum OpMode {iABC, iABx, iAsBx}; /* basic instruction format */

![](https://wyyhzc.gitbooks.io/gifs/content/opcode.jpg)

在lopcode.h中,枚举定义了Lua中的所有opcode以及紧跟着的注释注明了不同opcode所对应的格式.可以一边阅读一边结合着看.

还需要注意的一点是,在这里的A,B,Bx,sBx都只是表示的是操作数,但是具体到哪里去取值还需要查看注释中相关的说明,无非就是这么几种:

    以R开头表示从寄存器中取,但是其实在Lua中并没有寄存器这一概念,只有前面说的Lua栈,所以其实这里的寄存器指代的是Lua栈.

    UpValue表示从当前函数的Upval数组中取值.

    Kst开头的表示从常量数组中取值,常量数组的存放在Proto结构体的成员变量k中.

    Gbl开头的表示从全局变量表中取值,存放在LClosure结构体的env变量中.