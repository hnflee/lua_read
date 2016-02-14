# Lua5.1.4代码分析(九)-循环类操作

有了前面的基础,再来看循环类指令就很简单了,因为循环类指令,本质上可以改写成逻辑跳转类指令.

就以最简单的for循环来讲解循环类指令的实现.Lua中的for循环有两种形式,一种是平时常见的数字循环类,另一种是遍历数组,Hash的for循环语句.当然还是第一种更简单,所以就以数字循环类指令来讲解.

与数字for循环类指令相关的Opcode有两个:

```C
OP_FORPREP,/*	A sBx	R(A)-=R(A+2); pc+=sBx				*/

OP_FORLOOP,/*	A sBx	R(A)+=R(A+2);
		  	if R(A) less than R(A+1) then { pc+=sBx; R(A+3)=R(A) }*/
```
这两个指令是组合在一起使用的,其中FORPREP是循环开始的准备语句,而FORLOOP是循环判断跳转语句,我们还是以一个例子来说明:

```C
for i = 1, 10, 1 do
   print(i)
end
```
在实现时,Lua首先给几个循环判断相关的临时变量分配空间,依次将初始值,最终值,步长存放在R(A),R(A + 1),以及R(A + 2)中,而循环变量i则存放在R(A + 3)中.

于是可以根据此,将前面的for循环改写为以下的伪代码:

```
// OP_FORPREP指令部分
  _index = _index - _step;	// R(A)-=R(A+2);
  jmp for_prep;	    		// pc+=sBx

// 循环体
loop:
  print(i);

// OP_FORLOOP指令部分
for_prep:
  _index = _index + _step;	// R(A)+=R(A+2);
  if (_index < _limit) {	   // if R(A) less than R(A+1) then {
    jmp loop;  	       		   // pc+=sBx;
    i = _index;				// R(A+3)=R(A)
  }
```

剩下的难点,无非是将两个跳转位置的跳转点使用回填技术填充罢了.

这部分相关的代码,参考lparser.c中的fornum函数和forbody函数.

循环类指令还有几个,比如REPEAT,WHILE,原理差不多,仔细看看就明白了,不再详述.

