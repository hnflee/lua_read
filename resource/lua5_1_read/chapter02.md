# Lua5.1.4代码分析(二)字符串类型

Lua中字符串结构体的定义是:
```C
    typedef union TString {
      L_Umaxalign dummy;  /* ensures maximum alignment for strings */
      struct {
        CommonHeader;
        lu_byte reserved;
        unsigned int hash;
        size_t len;
      } tsv;
    } TString;
```

这里TString结构体是一个union, 最开始的L_Umaxalign dummy;起到的是对齐作用.紧跟着是CommonHeader,可以看出TString也是可GC数据类型的一种.

在Lua中,字符串是一个保存在一个全局的地方,在globale_state的strt里面,这是一个hash数组,专门用于存放字符串:

```C
    typedef struct stringtable {
      GCObject **hash;
      lu_int32 nuse;  /* number of elements */
      int size;
    } stringtable;
```

一个字符串TString,首先根据hash算法算出hash值,这就是stringtable中hash的索引值,如果这里已经有元素,则使用链表串接起来.

同时,TString中的字段reserved,表示这个字符串是不是保留字符串,比如Lua的关键字,在最开始赋值的时候是这么处理的:

```C
    void luaX_init (lua_State *L) {
      int i;
      for (i=0; itsv.reserved = cast_byte(i+1);  /* reserved word */
      }
    }
```

这里存放的值,是数组luaX_tokens中的索引:

```C
    const char *const luaX_tokens [] = {
        "and", "break", "do", "else", "elseif",
        "end", "false", "for", "function", "if",
        "in", "local", "nil", "not", "or", "repeat",
        "return", "then", "true", "until", "while",
        "..", "...", "==", ">=", "<=", "~=",
        "", "", "", "",
        NULL
    };
```

一方面可以迅速定位到是哪个关键字,另方面如果这个reserved字段不为0,则表示该字符串是不可自动回收的,在GC过程中会略过这个字符串的处理.

具体查找字符串时,首先计算出hash值,定位到所在的strt中的hash数组所在,再遍历hash桶所在链表,首先比较长度,如果相同再继续逐字节的比较字符串内容:

```C
    TString *luaS_newlstr (lua_State *L, const char *str, size_t l) {
      GCObject *o;
      unsigned int h = cast(unsigned int, l);  /* seed */
      size_t step = (l>>5)+1;  /* if string is too long, don't hash all its chars */
      size_t l1;
      for (l1=l; l1>=step; l1-=step)  /* compute hash */
        h = h ^ ((h<<5)+(h>>2)+cast(unsigned char, str[l1-1]));
      for (o = G(L)->strt.hash[lmod(h, G(L)->strt.size)];
           o != NULL;
           o = o->gch.next) {
        TString *ts = rawgco2ts(o);
        if (ts->tsv.len == l && (memcmp(str, getstr(ts), l) == 0)) {
          /* string may be dead */
          if (isdead(G(L), o)) changewhite(o);
          return ts;
        }
      }
      return newlstr(L, str, l, h);  /* not found */
    }
```