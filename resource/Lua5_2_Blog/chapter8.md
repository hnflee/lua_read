# 虚拟机指令(8) LOOP

Lua5.2种除了for循环之外，其他的各种循环都使用关系和逻辑指令，配合JMP指令来完成。

```
    local a = 0;  
    while(a < 10) do  
        a = a + 1;  
    end  
```

```
    1       [1]     LOADK           0 -1    ; 0  
    2       [2]     LT              0 0 -2  ; - 10  
    3       [2]     JMP             0 2     ; to 6  
    4       [3]     ADD             0 0 -3  ; - 1  
    5       [3]     JMP             0 -4    ; to 2  
    6       [4]     RETURN          0 1  
```

第二行使用LT对寄存器0和敞亮10进行比较，如果小于成立，跳过第三行的JMP，运行第四行的ADD指令，将a加1，然后运行第五行的JMP，跳转回第二行，重新判断条件。如果小于不成立，则直接运行下一个JMP指令，跳转到第六行结束。

对于for循环，Lua5.2使用了两套专门的指令，分别对应numeric for loop和generic for loop。

name |							 args |		   desc
------------ | ------------- | -------------
OP_FORLOOP|  A sBx |	     R(A)+=R(A+2);<br>if R(A) <?= R(A+1) then { pc+=sBx; R(A+3)=R(A) }
OP_FORPREP | A sBx |	     R(A)-=R(A+2); pc+=sBx

```
    local a;  
    for i = 1, 10 do  
        a = i;  
    end  
```

```
    main <test.lua:0,0> (8 instructions at 0x80048eb0)  
    0+ params, 5 slots, 1 upvalue, 5 locals, 2 constants, 0 functions  
            1       [1]     LOADNIL         0 0  
            2       [2]     LOADK           1 -1    ; 1  
            3       [2]     LOADK           2 -2    ; 10  
            4       [2]     LOADK           3 -1    ; 1  
            5       [2]     FORPREP         1 1     ; to 7  
            6       [3]     MOVE            0 4  
            7       [2]     FORLOOP         1 -2    ; to 6  
            8       [4]     RETURN          0 1  
    constants (2) for 0x80048eb0:  
            1       1  
            2       10  
    locals (5) for 0x80048eb0:  
            0       a       2       9  
            1       (for index)     5       8  
            2       (for limit)     5       8  
            3       (for step)      5       8  
            4       i       6       7  
    upvalues (1) for 0x80048eb0:  
            0       _ENV    1       0  
```

Numeric for loop内部使用了3个局部变量来控制循环，他们分别是"for index"，“for limit”和“for step”。“for index”用作存放初始值和循环计数器，“for limit”用作存放循环上限，“for step”用作存放循环步长。对于上面的程序，三个值分别是1，10和1。这三个局部变量对于使用者是不可见得，我们可以在生成代码的locals表中看到这3个局部变量，他们的有效范围为第五行道第八行，也就是整个for循环。还有一个使用到的局部变量，就是使用者自己指定的计数器，上例中为"i"。我们可以看到，这个局部变量的有效范围为6~7行，也就是循环的内部。这个变量在每次循环时都被设置成"for index"变量来使用。

上例中2~4行初始化循环使用的3个内部局部变量。第五行FORPREP用于准备这个循环，将for index减去一个for step，然后跳转到第七行。第七行的FORLOOP将for index加上一个for step，然后与for limit进行比较。如果小于等于for limit，则将i设置成for index，然后跳回第六行。否则就退出循环。我们可以看到，i并不用于真正的循环计数，而只是在每次循环时被赋予真正的计数器for index的值而已，所以在循环中修改i不会影响循环计数。

name |										 args |		  desc
------------ | ------------- | -------------
OP_TFORCALL |  A C |	     R(A+3), ... ,R(A+2+C) := R(A)(R(A+1), R(A+2));
OP_TFORLOOP |  A sBx |	     if R(A+1) ~= nil then { R(A)=R(A+1); pc += sBx }

```
    for i,v in 1,2,3 do  
        a = 1;  
    end  
```

```
    main <test.lua:0,0> (8 instructions at 0x80048eb0)  
    0+ params, 6 slots, 1 upvalue, 5 locals, 4 constants, 0 functions  
            1       [1]     LOADK           0 -1    ; 1  
            2       [1]     LOADK           1 -2    ; 2  
            3       [1]     LOADK           2 -3    ; 3  
            4       [1]     JMP             0 1     ; to 6  
            5       [2]     SETTABUP        0 -4 -1 ; _ENV "a" 1  
            6       [1]     TFORCALL        0 2  
            7       [1]     TFORLOOP        2 -3    ; to 5  
            8       [3]     RETURN          0 1  
    constants (4) for 0x80048eb0:  
            1       1  
            2       2  
            3       3  
            4       "a"  
    locals (5) for 0x80048eb0:  
            0       (for generator) 4       8  
            1       (for state)     4       8  
            2       (for control)   4       8  
            3       i       5       6  
            4       v       5       6  
    upvalues (1) for 0x80048eb0:  
            0       _ENV    1       0 
```

Generic for loop内部也使用了3个局部变量来控制循环，分别是"for generator”，“for state”和“for control”。for generator用来存放迭代使用的closure，每次迭代都会调用这个closure。for state和for control用于存放传给for generator的两个参数。Generic for loop还使用自定义的局部变量i，v，用来存储for generator的返回值。

上例中1~3行使用in后面的表达式列表(1,2,3)初始化3个内部使用的局部变量。第四行JMP调转到第六行。TFORCALL调用寄存器0(for generator)中的closure，传入for state和for control，并将结果返回给自定义局部变量列表i和v。第七行调用TFORLOOP进行循环条件判断，判断i是否为空。如果不为空，将i的值赋给for control，然后跳转到第五行，进行循环。

