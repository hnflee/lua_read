# 内部实现:TString

Lua使用TString结构体代表一个字符串对象。

``` C
    /* 
    ** Header for string value; string bytes follow the end of this structure 
    */  
    typedef union TString {  
      L_Umaxalign dummy;  /* ensures maximum alignment for strings */  
      struct {  
        CommonHeader;  
        lu_byte extra;  /* reserved words for short strings; "has hash" for longs */  
        unsigned int hash;  
        size_t len;  /* number of characters in string */  
      } tsv;  
    } TString; 
```


hash用来记录字符串对应的哈希值，len用来记录字符串长度。

这个结构体其实只是字符串对象的头数据，后面紧跟着字符窜内容和一个'\0'结尾。字符串"abc"对应的TString对象应该如下图所示

![](https://git.gitbook.com/raw/wyyhzc/gifs/master/1360054355_6651.png?token=d3l5aHpjOjkyODA1MGVkLTMxZDEtNDFmOS04MjY3LWU1YzdmNjU4M2U3Nw%3D%3D)

字符串对象的最终大小应该是TString的大小+字符串长度+1。

TString对应的lua对象类型为LUA_TSTRING。根据字符串长度（luaconf.h中的LUAI_MAXSHORTLEN，默认为40）的不同，TString对象还被分成两种子类型：LUA_TSHRSTR（短字符串）和LUA_TLNGSTR（长字符串）。

对于短字符串，在实际使用中一般用来作为索引或者需要进行字符串比较。不同于其他的对象，Lua并不是将其连接到全局的allgc对象链表上，而是将其放到全局状态global_State中的字符串表中进行管理。这个字符串表是一个stringtable类型的全局唯一的哈希表。当需要创建一个短字符串对象时，会首先在这个表中查找已有对象。所有的短字符串都是全局唯一的，不会存在两个相同的短字符串对象。如果需要比较两个短字符串是否相等，只需要看他们指向的是否是同一个TString对象就可以了，速度非常快。如果短字符串对象的extra > 0，表示这是一个系统保留的字符串。extra的值直接对应着词法分析时的一个token值，这样可以加速词法分析的速度。

对于长字符串，与短字符串相反，在实际使用中一般只是用来存储文本数据，很少需要比较或者索引。所以长字符串直接被挂接到allgc链表上当作普通的对象来处理。Lua不会对新创建的长字符串对象计算哈希值，也不保证长字符串对象的唯一性。当长字符串需要被用来当作索引时，会为其计算一次哈希值，并使用extra来记录是否已经为其计算了哈希值。

计算字符串的哈希值需要遍历字符串的每一个字符，如果字符串很长，会非常影响效率。对于大于等于32的字符串，Lua不会遍历每个字符，而是按照一定的间隔获取字符，最多遍历31次。这使得字符串的哈希值计算与字符串长度无关。
如果要将Lua部署到web上面来处理大量的基于文本的http请求，文本处理的安全性就变得尤为重要。为了防止Hash DoS攻击，字符串哈希值生成需要一个seed，这个seed是每次启动Lua时随机生成的。这就使不同的Lua实例对于同一个字符串会有不同的哈希值。 