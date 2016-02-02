# 虚拟机指令(6）FUNCTION 

name|args|desc
------------ | ------------- | -------------
OP_CALL|A B C|R(A), ... ,R(A+C-2) := R(A)(R(A+1), ... ,R(A+B-1))
 	    
CALL执行一个函数调用。寄存器A中存放函数对象，所有参数按顺序放置在A后面的寄存器中。B－1表示参数个数 。如果参数列表的最后一个表达式是变长的，则B会设置为0，表示使用A＋1到当前栈顶作为参数。函数调用的返回值会按顺序存放在从寄存器A开始的C-1个寄存器中。如果C为0,表示返回值的个数由函数决定。

```
    f();  
```

```
    1   [1] GETTABUP    0 0 -1  ; _ENV "f"  
    2   [1] CALL        0 1 1  
    3   [1] RETURN      0 1 
```
第一行取得全局变量f的值，保存到寄存器0。第二行CALL调用寄存器0中的函数，参数和返回值都是0。

```
    local t = {f(...)}; 
```

```
    1   [1] NEWTABLE    0 0 0  
    2   [1] GETTABUP    1 0 -1  ; _ENV "f"  
    3   [1] VARARG      2 0  
    4   [1] CALL        1 0 0  
    5   [1] SETLIST     0 0 1   ; 1  
    6   [1] RETURN      0 1  
```

第一行NETTABLE创建一个表放到寄存器0中。第二行获取全局变量f放到寄存器1中。第三行VARARG表示使用当前函数的变长参数列表。第四行的CALL调用寄存器1中的函数，B为0，代表参数是变长的。前面讲过，如果表的构造的最后一项是多返回值的表达式，则这个表会接受所有的返回值。这里就是这种情况，表的构造会接受函数所有的返回值，所以C也为0。

name|args|desc
------------ | ------------- | -------------
OP_TAILCALL|A B C|return R(A)(R(A+1), ... ,R(A+B-1))

如果一个return statement只有一个函数调用表达式，这个函数调用指令CALL会被改为TAILCALL指令。TAILCALL不会为要调用的函数增加调用堆栈的深度，而是直接使用当前调用信息。ABC操作数与CALL的意思一样，不过C永远都是0。TAILCALL在执行过程中，只对lua closure进行tail call处理，对于c closure，其实与CALL没什么区别。

```
    return f();  
```

```
    1   [1] GETTABUP    0 0 -1  ; _ENV "f"  
    2   [1] TAILCALL    0 1 0  
    3   [1] RETURN      0 0  
    4   [1] RETURN      0 1  
```

上面如果f是一个lua closure，那么执行到第二行后，此函数就会返回了，不会执行到后面第三行的RETURN。如果f是一个c closure，那就和CALL一样调用这个函数，然后依赖第三行的RETURN返回。这就是为什么TAILCALL后面还会己跟着生成一个RETURN的原因。


name|args|desc
------------ | ------------- | -------------
OP_RETURN|A B |return R(A), ... ,R(A+B-2)

RETURE将返回结果存放到寄存器A到寄存器A＋B－2中。如果返回的为变长表达式，则B会被设置为0，表示将寄存器A到当前栈顶的所有值返回。

```
    return 1; 
```

```
    1   [1] LOADK       0 -1    ; 1  
    2   [1] RETURN      0 2  
    3   [1] RETURN      0 1 
```
RETURN只能从寄存器返回数据，所以第一行LOADK先将常量1装载道寄存器0，然后返回。

```
    return ...;  
```

```
    1   [1] VARARG      0 0  
    2   [1] RETURN      0 0  
    3   [1] RETURN      0 1
```
因为返回的为变长表达式，B为0。 

name|args|desc
------------ | ------------- | -------------
OP_CLOSURE|A Bx |R(A) := closure(KPROTO[Bx])

CLOSURE为指定的函数prototype创建一个closure，并将这个closure保存到寄存器A中。Bx用来指定函数prototype的id。

```
    local function f()  
    end  
```

```
    main <test.lua:0,0> (2 instructions at 0x102a016f0)  
    0+ params, 2 slots, 1 upvalue, 1 local, 0 constants, 1 function  
        1   [2] CLOSURE     0 0 ; 0x102a019b0  
        2   [2] RETURN      0 1  
    constants (0) for 0x102a016f0:  
    locals (1) for 0x102a016f0:  
        0   f   2   3  
    upvalues (1) for 0x102a016f0:  
        0   _ENV    1   0  
      
    function <test.lua:1,2> (1 instruction at 0x102a019b0)  
    0 params, 2 slots, 0 upvalues, 0 locals, 0 constants, 0 functions  
        1   [2] RETURN      0 1  
    constants (0) for 0x102a019b0:  
    locals (0) for 0x102a019b0:  
    upvalues (0) for 0x102a019b0:
```

上面生成了一个主函数和一个子函数，CLOSURE将为这个索引为0的子函数生成一个closure，并保存到寄存器0中。 

name|args|desc
------------ | ------------- | -------------
OP_VARARG|A B |R(A), R(A+1), ..., R(A+B-2) = vararg 

VARARG直接对应'...'运算符。VARARG拷贝B-1个参数到从A开始的寄存器中，如果不足，使用nil补充。如果B为0，表示拷贝实际的参数数量。

```
    local a = ...;  
```
```
    1   [1] VARARG      0 2  
    2   [1] RETURN      0 1 
```

上面第一行表示拷贝B-1个，也就是1个变长参数到寄存器0,也就是local a中。 

```
    f(...); 
```
```
    1   [1] GETTABUP    0 0 -1  ; _ENV "f"  
    2   [1] VARARG      1 0  
    3   [1] CALL        0 0 1  
    4   [1] RETURN      0 1 
```
由于函数调用最后一个参数可以接受不定数量的参数，所以第二行生成的VARARG的B参数为0。

name|args|desc
------------ | ------------- | -------------
OP_SELF|A B C|R(A+1) := R(B); R(A) := R(B)[RK(C)] 

SELF是专门为“:”运算符准备的指令。从寄存器B表示的table中，获取出C作为key的closure，存入寄存器A中，然后将table本身存入到寄存器A＋1中，为接下来调用这个closure做准备。

```
    a:b(); 
```
```
    1   [1] GETTABUP    0 0 -1  ; _ENV "a"  
    2   [1] SELF        0 0 -2  ; "b"  
    3   [1] CALL        0 2 1  
    4   [1] RETURN      0 1  
```

看一下与上面语法等价的表示方法生成的指令： 

```
    a.b(a);
```
```
    1   [1] GETTABUP    0 0 -1  ; _ENV "a"  
    2   [1] GETTABLE    0 0 -2  ; "b"  
    3   [1] GETTABUP    1 0 -1  ; _ENV "a"  
    4   [1] CALL        0 2 1  
    5   [1] RETURN      0 1  
```
比使用“:"操作符多使用了一个指令。所以，如果需要使用这种面向对象调用的语义时，应该尽量使用”:"。
