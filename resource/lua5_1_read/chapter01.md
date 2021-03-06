# Lua5.1.4代码分析(一)Lua的通用数据类型

TValue这个结构体是Lua的通用结构体,顾名思义,它存放的是Lua的数据,既然是”通用”的,同时也说明,Lua中的所有的数据都可以使用这个结构体来表示.很容易想到,在面向对象中,这个结构体是一个基类,派生出来的都是其他的子类.如果使用C语言来模拟,就是定义出一个通用的结构体作为”父类”,然后子类的结构体中以这个父类作为结构体的第一个成员变量.比如这样:
```C
    struct Common {
       // ....
    };
    struct Object1 {
      struct Common com;
      // ...
    };

```

但是Lua的做法稍微不一样,来看看Lua是如何做的.

TValue结构体内部有几个宏,为了方便,把它们展开之后就是这样的:

```C
    typedef struct lua_TValue {
    	    union {
    	      union GCObject {
    	      	    struct GCheader {
    		    	     GCObject *next; lu_byte tt; lu_byte marked;
    			     	      } gch;
    
		union TString ts;
    		      union Udata u;
    		      	    union Closure cl;
    			    	  struct Table h;
    				  	 struct Proto p;
    					 	struct UpVal uv;
    						       struct lua_State th;  /* thread */
    						         } gc;
    
	  void *p;
    	    lua_Number n;
    	      int b;
    	      } value;
    	      int tt;
    } TValue;
```

这个结构体定义,总体来说分为两个部分:tt存放的数据类型,而value域存放的是各种数据.而在其中,又划分为两个部分,可gc的数据类型使用union放在一起,剩下的就是不可gc的数据类型了:void*,lua_Number,int.

再详细看看gc union的定义,可以看到各种可gc的类型(Tstring,Udata..etc)和一个GCHeader放在一起,也就是说,当这部分还是数据的时候,数据部分启用,否则就是gc部分了.这里的GCHeader包括了三个部分:next指针将可gc的数据串联成链表,tt表示数据类型,marked存放的gc处理时的颜色值.
这是另一种方式的使用C语言实现的面向对象,对外部而言,TValue结构体可以看作是”基类”,真正进行处理时,再根据数据类型决定到底使用value union中的哪个数据部分.可以看到lua源代码中定义了很多宏就是这样操作Tvalue数据指针的,比如:

```C
    #define hvalue(o)	check_exp(ttistable(o), &(o)->value.gc->h)  
```
这个宏定义了如何从TValue指针得到Table结构体:首先判断数据类型是Table,然后将value的gc union中Table *h取出.

反之,要从一个具体的类型转换再赋值为相应的TValue,Lua源代码中也提供了相应的宏.因为TValue结构体的中的value域是一个union,所以其实随便强制转换为其中的哪一种类型都可以,不过看上去最舒服的写法还是直接转换为公共类型GCObject了,比如:

```C
    #define setsvalue(L,obj,x) \
      { TValue *i_o=(obj); \
        i_o->value.gc=cast(GCObject *, (x)); i_o->tt=LUA_TSTRING; \
    checkliveness(G(L),i_o); }
```

故事到这里还没有完.注意到一点:

```C
    union GCObject {
      GCheader gch;
      union TString ts;
      union Udata u;
      union Closure cl;
      struct Table h;
      struct Proto p;
      struct UpVal uv;
      struct lua_State th;  /* thread */
    };
```
其中的GCheader展开是这样的:

```C
    typedef struct GCheader {
      CommonHeader;
    } GCheader;
```

而随便抽在GCObject结构体中的数据类型结构体定义,都发现也包含了一个CommonHeader结构体,比如:

```C
    typedef struct Table {
      CommonHeader;
      lu_byte flags;
      lu_byte lsizenode;  /* log2 of size of `node' array */
      struct Table *metatable;
      TValue *array;  /* array part */
      Node *node;
      Node *lastfree;  /* any free position is before this position */
      GCObject *gclist;
      int sizearray;  /* size of `array' array */
    } Table;
```

换言之,在GCObject中,无论是哪个数据结构体,都自己有一份CommonHeader.仔细观察,其实GCObject这个union的内存分布,最开始部分无论如何都是留给CommonHeader的(部分结构体做了对齐,比如TString,具体定义可以自己去看.).这样做,就保证了一个存放在TValue结构体中的数据,既可以使用CommonHeader关于GC的部分,也可以使用到自己本身的数据部分了.