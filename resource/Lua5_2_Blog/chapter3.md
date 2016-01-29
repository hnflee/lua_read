# 虚拟机指令(3) Upvalues & Globals


在编译期，如果要访问变量a时，会依照以下的顺序决定变量a的类型：

1. a是当前函数的local变量
2. a是外层函数的local变量，那么a是当前函数的upvalue
3. a是全局变量


local变量本身就存在于当前的register中，所有的指令都可以直接使用它的id来访问。而对于upvalue，lua则有专门的指令负责获取和设置。

全局变量在lua5.1中也是使用专门的指令，而5.2对这一点做了改变。Lua5.2中没有专门针对全局变量的指令，而是把全局表放到最外层函数的名字为"_ENV"的upvalue中。对于全局变量a，相当于编译期帮你改成了_ENV.a来进行访问。

| name | args | desc |
| -- | -- | -- |
| OP_GETUPVAL | A B C  | R(A) := UpValue[B] |
| OP_SETUPVAL | A B  | UpValue[B] := R(A) |
| OP_GETTABUP | A B C | R(A) := UpValue[B][RK(C)] |
| OP_SETTABUP | A B C | UpValue[A][RK(B)] := RK(C) |



GETUPVAL将B为索引的upvalue的值装载到A寄存器中。SETUPVAL将A寄存器的值保存到B为索引的upvalue中。

GETTABUP将B为索引的upvalue当作一个table，并将C做为索引的寄存器或者常量当作key获取的值放入寄存器A。SETTABUP将A为索引的upvalue当作一个table，将C寄存器或者常量的值以B寄存器或常量为key，存入table。

```
    local u = 0;  
    function f()   
        local l;  
        u = 1;   
        l = u;  
        g = 1;  
        l = g;  
    end  
```

```
    main <test.lua:0,0> (4 instructions at 0x80048eb0)  
    0+ params, 2 slots, 1 upvalue, 1 local, 2 constants, 1 function  
        1   [1] LOADK       0 -1    ; 0  
        2   [8] CLOSURE     1 0 ; 0x80049140  
        3   [2] SETTABUP    0 -2 1  ; _ENV "f"  
        4   [8] RETURN      0 1  
    constants (2) for 0x80048eb0:  
        1   0  
        2   "f"  
    locals (1) for 0x80048eb0:  
        0   u   2   5  
    upvalues (1) for 0x80048eb0:  
        0   _ENV    1   0  
      
    function <test.lua:2,8> (7 instructions at 0x80049140)  
    0 params, 2 slots, 2 upvalues, 1 local, 2 constants, 0 functions  
        1   [3] LOADNIL     0 0  
        2   [4] LOADK       1 -1    ; 1  
        3   [4] SETUPVAL    1 0 ; u  
        4   [5] GETUPVAL    0 0 ; u  
        5   [6] SETTABUP    1 -2 -1 ; _ENV "g" 1  
        6   [7] GETTABUP    0 1 -2  ; _ENV "g"  
        7   [8] RETURN      0 1  
    constants (2) for 0x80049140:  
        1   1  
        2   "g"  
    locals (1) for 0x80049140:  
        0   l   2   8  
    upvalues (2) for 0x80049140:  
        0   u   1   0  
        1   _ENV    0   0  
        
```

上面的代码片段生成一个主函数和一个内嵌函数。根据前面说到的变量规则，在内嵌函数中，l是local变量，u是upvalue，g由于既不是local变量，也不是upvalue，当作全局变量处理。我们先来看内嵌函数，生成的指令从17行开始。

第17行的LOADNIL前面已经讲过，为local变量赋值。下面的LOADK和SETUPVAL组合，完成了u = 1。因为1是一个常量，存在于常量表中，而lua没有常量与upvalue的直接操作指令，所以需要先把常量1装在到临时寄存器1种，然后将寄存器1的值赋给upvalue 0，也就是u。第20行的GETUPVAL将upvalue u赋给local变量l。第21行开始的SETTABUP和GETTABUP就是前面提到的对全局变量的处理了。g=1被转化为_ENV.g=1。_ENV是系统预先设置在主函数中的upvalue，所以对于全局变量g的访问被转化成对upvalue[_ENV][g]的访问。SETTABUP将upvalue 1(_ENV代表的upvalue)作为一个table，将常量表2（常量"g"）作为key的值设置为常量表1（常量1）；GETTABUP则是将upvalue 1作为table，将常量表2为key的值赋给寄存器0（local l）。


 										   
											
												