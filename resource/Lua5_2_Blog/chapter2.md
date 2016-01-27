# 虚拟机指令(2) MOVE & LOAD

| name | args | desc |
| -- | -- | -- |
| OP_MOVE | A B | R(A) := R(B) |

OP_MOVE用来将寄存器B中的值拷贝到寄存器A中。由于Lua是register based vm，大部分的指令都是直接对寄存器进行操作，而不需要对数据进行压栈和弹栈，所以需要OP_MOVE指令的地方并不多。最直接的使用之处就是将一个local变量复制给另一个local变量时:

```    
    local a;  
    local b = a;
```

```    
    1   [1] LOADNIL     0 0  
    2   [2] MOVE        1 0  
    3   [2] RETURN      0 1  
```    

在编译过程中，Lua会将每个local变量都分配到一个指定的寄存器中。在运行期，lua使用local变量所对应的寄存器id来操作local变量，而local变量的名字除了提供debug信息外，没有其他作用。

在这里a被分配给register 0，b被分配给register 1。第二行的MOVE表示将a(register 0)的值赋给b(register 1)。其他使用的地方基本都是对寄存器的位置有特殊要求的地方，比如函数参数的传递等等。


 | name | args | desc |
| -- | -- | -- |
| OP_LOADK | A Bx | R(A) := Kst(Bx) |		

LOADK将Bx表示的常量表中的常量值装载到寄存器A中。很多其他指令，比如数学操作指令，其本身可以直接从常量表中索引操作数，所以可以不依赖于LOADK指令。

```
    local a=1;  
    local b="foo";  
```
```
    1   [1] LOADK       0 -1    ; 1  
    2   [2] LOADK       1 -2    ; "foo"  
    3   [2] RETURN      0 1  
    onstants (2) for 0x80048eb0:  
    1   1  
    2   "foo"   
```
 | name | args | desc |
| -- | -- | -- |
| OP_LOADKX | A  | R(A) := Kst(extra arg) |	

LOADKX是lua5.2新加入的指令。当需要生成LOADK指令时，如果需要索引的常量id超出了Bx所能表示的有效范围，那么就生成一个LOADKX指令，取代LOADK指令，并且接下来立即生成一个EXTRAARG指令，并用其Ax来存放这个id。5.2的这个改动使得一个函数可以处理超过262143个常量。

 | name | args | desc |
| -- | -- | -- |
| OP_LOADBOOL | A B C  | R(A) :=(Bool)B; if (C) pc++ |	

LOADBOOL将B所表示的boolean值装载到寄存器A中。B使用0和1分别代表false和true。C也表示一个boolean值，如果C为1，就跳过下一个指令。

```
    local a = true;  
```
```
    1   [1] LOADBOOL    0 1 0  
    2   [1] RETURN      0 1  
```

C在这里的作用比较特殊。要了解C的具体用处，首先要知道lua中对于逻辑和关系表达式是如何处理的，比如： 

```
    local a = 1 < 2  
```

对于上面的代码，一般我们会认为lua应该先对1<2求出一个boolean值，然后放入到a中。然而实际上产生出来的代码为： 

```
    1   [1] LT          1 -1 -2 ; 1 2  
    2   [1] JMP         0 1 ; to 4  
    3   [1] LOADBOOL    0 0 1  
    4   [1] LOADBOOL    0 1 0  
    5   [1] RETURN      0 1  
    onstants (2) for 0x80048eb0:  
    1   1  
    2   2   
```
可以看到，lua生成了LT和JMP指令，另外再加上两个LOADBOOL对于a赋予不同的boolean值。LT（后面会详细讲解）指令本身并不产生一个boolean结果值，而是配合后面紧跟的JMP实现true和false的不同跳转。如果LT评估为true，就继续执行，也就是执行到JMP，然后调转到4，对a赋予true；否则就跳过下一条指令到达第三行，对a赋予false，并且跳过下一个指令。所以上面的代码实际的意思被转化为：
```
    local a;  
    if 1 < 2 then  
        a = true;  
    else  
        a = false;  
    end 
```

逻辑或者关系表达式之所以被设计成这个样子，主要是为if语句和循环语句所做的优化。不用将整个表达式估值成一个boolean值后再决定跳转路径，而是评估过程中就可以直接跳转，节省了很多指令。

C的作用就是配合这种使用逻辑或关系表达式进行赋值的操作，他节省了后面必须跟的一个JMP指令。

 | name | args | desc |
| -- | -- | -- |
| OP_LOADNIL | A B   | R(A), R(A+1), ..., R(A+B) := nil |

LOADNIL将使用A到B所表示范围的寄存器赋值成nil。用范围表示寄存器主要为了对以下情况进行优化：

```
    local a,b,c;  
```

```
    1   [1] LOADNIL     0 2  
    2   [1] RETURN      0 1  
```

对于连续的local变量声明，使用一条LOADNIL指令就可以完成，而不需要分别进行赋值。

对于一下情况

```
    local a;  
    local b = 0;  
    local c; 
```
```
    1   [1] LOADNIL     0 0  
    2   [2] LOADK       1 -1    ; 0  
    3   [3] LOADNIL     2 0  
```
在Lua5.2中，a和c不能被合并成一个LOADNIL指令。所以以上写法理论上会生成更多的指令，应该予以避免，而改写成 
```
    local a,c;  
    local b = 0;  
```