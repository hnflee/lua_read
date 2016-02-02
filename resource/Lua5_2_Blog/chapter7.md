# 虚拟机指令(7) 关系和逻辑指令

name |		args |	desc
------------ | ------------- | -------------
OP_JMP |     A sBx |	     pc+=sBx; if (A) close all upvalues >= R(A) + 1

JMP执行一个跳转，sBx表示跳转的偏移位置，被加到当前指向下一指令的指令指针上。如果sBx为0，表示没有任何跳转；1表示跳过下一个指令；-1表示重新执行当前指令。如果A>0，表示需要关闭所有从寄存器A+1开始的所有local变量。实际执行的关闭操作只对upvalue有效。

JMP最直接的使用就是对应lua5.2新加入的goto语句：

```
    ::l::  
    goto l; 
```

```
    1       [1]     JMP             0 -1    ; to 1  
    2       [2]     RETURN          0 1  
```

这是一个无限循环。第一行JMP的sBx为-1，表示重新执行JMP。

```
    do  
        local a;  
        function f() a = 1 end  
    end  
```

```
    main <test.lua:0,0> (5 instructions at 0x80048eb0)  
    0+ params, 2 slots, 1 upvalue, 1 local, 1 constant, 1 function  
            1       [2]     LOADNIL         0 0  
            2       [3]     CLOSURE         1 0     ; 0x80049128  
            3       [3]     SETTABUP        0 -1 1  ; _ENV "f"  
            4       [3]     JMP             1 0     ; to 5  
            5       [4]     RETURN          0 1  
    constants (1) for 0x80048eb0:  
            1       "f"  
    locals (1) for 0x80048eb0:  
            0       a       2       5  
    upvalues (1) for 0x80048eb0:  
            0       _ENV    1       0  
      
    function <test.lua:3,3> (3 instructions at 0x80049128)  
    0 params, 2 slots, 1 upvalue, 0 locals, 1 constant, 0 functions  
            1       [3]     LOADK           0 -1    ; 1  
            2       [3]     SETUPVAL        0 0     ; a  
            3       [3]     RETURN          0 1  
    constants (1) for 0x80049128:  
            1       1  
    locals (0) for 0x80049128:  
    upvalues (1) for 0x80049128:  
            0       a       1       0 
```

上面的代码在do block中创建了一个局部变量a，并且a作为upvalue在函数f中被引用到。到退出do block是，a会退出他的有效域，并且关闭他对应的upvalue。Lua5.2中去除了以前专门处理关闭upvalue的指令CLOSE，而把这个功能加入到了JMP中。所以，生成的指令第四行的JMP在这里没有执行跳转，而只是为了关闭a的upvalue。

JMP其他的功能就是配合逻辑和关系指令（统称为test指令），实现程序的条件跳转。每个test辑指令与JMP搭配，都会将接下来生成的指令分为两个集合，满足条件的为true集合，否则为false集合。当test条件满足时，指令指针回+1，跳过后面紧跟的JMP指令，然后继续执行。当test条件不满足时，则继续执行，也就到了JMP，然后跳转到分支代码。

![](https://git.gitbook.com/raw/wyyhzc/-lua5-2-/master/wyyhzc/1358221143_7162.png?token=d3l5aHpjOjkyODA1MGVkLTMxZDEtNDFmOS04MjY3LWU1YzdmNjU4M2U3Nw%3D%3D)

name |	args |	desc
------------ | ------------- | -------------
OP_EQ |	     A B C |	     if ((RK(B) == RK(C)) ~= A) then pc++
OP_LT |	     A B C |	     if ((RK(B) <  RK(C)) ~= A) then pc++
OP_LE |	     A B C |	     if ((RK(B) <= RK(C)) ~= A) then pc++

关系指令对RK(B)和RK(C)进行比较，然后将比较结果与A指定的boolean值进行比较，来决定最终的boolean值。A在这里为每个关系指令提供了两种比较目标，满足和不满足。比如OP_LT何以用来实现“<”和“>”。

```
    local a,b,c;  
    a = b < c;  
```

```
    1       [1]     LOADNIL         0 2  
    2       [2]     LT              1 1 2  
    3       [2]     JMP             0 1     ; to 5  
    4       [2]     LOADBOOL        0 0 1  
    5       [2]     LOADBOOL        0 1 0  
    6       [2]     RETURN          0 1 
```

第二行的LT对寄存器1和2进行LT比较，如果结果为true，则继续执行后面的JMP，跳转到第五行的LOADBOOL，将寄存器0赋值为true；如果结果为false，则跳过后面的JMP，执行第四行的LOADBOOL，将寄存器0赋值为false。我们前面讲过关于LOADBOOL，第四行执行后会跳过第五行的赋值。

name |	args |	desc
------------ | ------------- | -------------
OP_TEST |    A C |	     if not (R(A) <=> C) then pc++
OP_TESTSET | A B C |	     if (R(B) <=> C) then R(A) := R(B) else pc++

逻辑指令用于实现and和or逻辑运算符，或者在条件语句中判断一个寄存器。TESTSET将寄存器B转化成一个boolean值，然后与C进行比较。如果不相等，跳过后面的JMP指令。否则将寄存器B的值赋给寄存器A，然后继续执行。TEST是TESTSET的简化版，不需要赋值操作。

```
    local a,b,c;  
    a = b and c;  
```

```
    1       [1]     LOADNIL         0 2  
    2       [2]     TESTSET         0 1 0  
    3       [2]     JMP             0 1     ; to 5  
    4       [2]     MOVE            0 2  
    5       [2]     RETURN          0 1  
```

第二行的TESTSET将寄存器1的值与false比较。如果不成立，跳过JMP，执行第四行的MOVE，将寄存器2的值赋给寄存器0。否则，将寄存器1的值赋给寄存器0；然后执行后面的JMP。

上面的代码等价于

```
    if b then  
     a = c  
    else  
     a = b  
    end 
```