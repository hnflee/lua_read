# 虚拟机指令(5）Arithmetic

 name | args | desc 
------------ | ------------- | ------------- 
 OP_ADD | A B C | R(A) := RK(B) + RK(C) 
 OP_SUB | A B C | R(A) := RK(B) - RK(C) 
 OP_MUL | A B C | R(A) := RK(B) * RK(C) 
 OP_DIV | A B C | R(A) := RK(B) / RK(C) 
 OP_MOD | A B C | R(A) := RK(B) % RK(C) 
 OP_POW | A B C | R(A) := RK(B) ^ RK(C) 

   	 
上表中的指令都是与lua本身的二元操作符一一对应的标准3地址指令。B和C两个操作数计算的结果存入A中。

```
    local a = 1;  
    a = a + 1;  
    a = a - 1;  
    a = a * 1;  
    a = a / 1;  
    a = a % 1;  
    a = a ^ 1;

```

```
    main <test.lua:0,0> (8 instructions at 0x80048eb0)
    0+ params, 2 slots, 1 upvalue, 1 local, 1 constant, 0 functions
       1       [1]	LOADK      0 -1	    ; 1
       2       [2]	ADD        0 0 -1   ; - 1
       3       [3]	SUB        0 0 -1   ; - 1
       4       [4]	MUL        0 0 -1   ; - 1
       5       [5]	DIV        0 0 -1   ; - 1
       6       [6]	MOD        0 0 -1
       7       [7]	POW        0 0 -1	; - 1
       8       [7]	RETURN     0 1
    constants (1) for 0x80048eb0:
    	      1	  1
    locals (1) for 0x80048eb0:
    	   0   a   2	9
    upvalues (1) for 0x80048eb0:
    	     0	 _ENV	1	0  
```

| name | args | desc |
| -- | -- | -- |
| OP_UNM | A B  | R(A) := -R(B) |
| OP_NOT | A B  | R(A) := not R(B) |

```
    local a = 1;  
    local b = not a;  
    local c = -a; 
```

```
    1   [1] LOADK       0 -1    ; 1  
    2   [2] NOT         1 0  
    3   [3] UNM         2 0  
    4   [3] RETURN      0 1   
```

在编译和指令生成阶段，lua还支持所有一元和二元操作符表达式的常量表达式折叠”（const expression folding）优化。也就是如果计算操作数如果都是数字常量，可以在编译期计算出结果，就直接使用这个结果值，而不用生成计算指令。

```
    local a = 1 + 1;  
    local b = not 1; 
```
```
    1   [1] LOADK       0 -1    ; 2  
    2   [2] LOADBOOL    1 0 0  
    3   [2] RETURN      0 1  
```

从生成的结果可以看到1+1并没有生成对应的OP_ADD，而是直接把结果2赋值给了a。并且也没有为not 1生成OP_NOT指令，而是直接将false赋值给了b。 


| name | args | desc |
| -- | -- | -- |
| OP_LEN | A B  | R(A) := length of R(B) |

LEN直接对应'#'操作符，返回B对象的长度，并保存到A中。

```
    local a = #"foo"; 
```
```
    1   [1] LOADK       0 -1    ; "foo"  
    2   [1] LEN         0 0  
    3   [1] RETURN      0 1
```

| name | args | desc |
| -- | -- | -- |
| OP_CONCAT | A B C  | R(A) := R(B).. ... ..R(C) |

CONCAT将B和C指定范围内的字符串按顺序传接到一起，将结果存入到A。

```
    local a = "foo1".."foo2".."foo3"; 
```

```
    1   [1] LOADK       0 -1    ; "foo1"  
    2   [1] LOADK       1 -2    ; "foo2"  
    3   [1] LOADK       2 -3    ; "foo3"  
    4   [1] CONCAT      0 0 2  
    5   [1] RETURN      0 1
```