# 内部实现:Garbage Collection(2)


### **GCObject**



Lua使用union GCObject来表示所有的垃圾回收对象：

```C

 /* 
 ** Union of all collectable objects 
 */  
 union GCObject {  
   GCheader gch;  /* common header */  
   union TString ts;  
   union Udata u;  
   union Closure cl;  
   struct Table h;  
   struct Proto p;  
   struct UpVal uv;  
   struct lua_State th;  /* thread */  
 }; 

```

这就相当于在C++中，将所有的GC对象从GCheader派生，他们都共享GCheader。

```C
     /* 
     ** Common Header for all collectable objects (in macro form, to be 
     ** included in other objects) 
     */  
     #define CommonHeader    GCObject *next; lu_byte tt; lu_byte marked  
       
       
     /* 
     ** Common header in struct form 
     */  
     typedef struct GCheader {  
       CommonHeader;    
     } GCheader;  
    
```

marked这个标志用来记录对象与GC相关的一些标志位。其中0和1位用来表示对象的white状态和垃圾状态。当垃圾回收的标识阶段结束后，剩下的white对象就是垃圾对象。由于lua并不是立即清除这些垃圾对象，而是一步步逐渐清除，所以这些对象还会在系统中存在一段时间。这就需要我们能够区分出同样为white状态的垃圾对象和非垃圾对象。Lua使用两个标志位来表示white，就是为了高效的解决这个问题。这个标志位会轮流被当作white状态标志，另一个表示垃圾状态。在global_State中保存着一个currentwhite，来表示当前是那个标志位用来标识white。每当GC标识阶段完成，系统会切换这个标志位，这样原来为white的所有对象不需要遍历就变成了垃圾对象，而真正的white对象则使用新的标志位标识。

第2个标志位用来表示black状态，而既非white也非black就是gray状态。

除了short string和open upvalue之外，所有的GCObject都通过next被串接到全局状态global_State中的allgc链表上。我们可以通过遍历allgc链表来访问系统中的所有GCObject。short string被字符串标单独管理。open upvalue会在被close时也连接到allgc上。


### **引用关系**

垃圾回收过程通过对象之间的引用关系来标识对象。以下是lua对象之间在垃圾回收标识过程中需要遍历的引用关系：

![](https://git.gitbook.com/raw/wyyhzc/gifs/master/1365325183_1984.png?token=d3l5aHpjOjkyODA1MGVkLTMxZDEtNDFmOS04MjY3LWU1YzdmNjU4M2U3Nw%3D%3D)

TSTRING字符串对象，无论是长串还是短串，都没有对其他对象的引用。

usedata对象会引用到一个metatable和一个env table。

Upval对象通过v引用一个TValue，再通过这个TValue间接引用一个对象。在open状态下，这个v指向stack上的一个TValue。在close状态下，v指向Upval自己的TValue。

Table对象会通过key，value引用到其他对象，并且如果数组部分有效，也会通过数组部分引用。并且，table会引用一个metatable对象。

Lua closure会引用到Proto对象，并且会通过upvalues数组引用到Upval对象。

C closure会通过upvalues数组引用到其他对象。这里的upvalue与lua closure的upvalue完全不是一个意思。

Proto对象会引用到一些编译期产生的名称，常量，以及内嵌于本Proto中的Proto对象。

Thread对象通过stack引用其他对象。


### **barrier**


在《原理》中我们说过，incremental gc在mark阶段，为了保证“所有的black对象都不会引用white对象”这个不变性，需要使用barrier。

barrier被分为“向前”和“向后”两种。

luaC_barrier_函数用来实现“向前”的barrier。“向前”的意思就是当一个black对象需要引用一个white对象时，立即mark这个white对象。这样white对象就变为gray对象，等待下一步的扫描。这也就是帮助gc向前标识一步。luaC_barrier_函数被用在以下引用变化处：

1.     虚拟机执行过程中或者通过api修改close upvalue对其他对象的引用
2.     通过api设置userdata或table的metatable引用
3.     通过api设置userdata的env table引用
4.     编译构建proto对象过程中proto对象对其他编译产生对象的引用

luaC_barrierback_函数用来实现“向后”的barrier。“向后”的意思就是当一个black对象需要引用一个white对象时，将已经扫描过的black对象再次变为gray对象，等待重新扫描。这也就是将gc的mark后退一步。luaC_barrierback_目前只用于监控table的key和value对象引用的变化。Table是lua中最主要的数据结构，连全局变量都是被保存在一个table中，所以table的变化是比较频繁的，并且同一个引用可能被反复设置成不同的对象。对table的引用使用“向前”的barrier，逐个扫描每次引用变化的对象，会造成很多不必要的消耗。而使用“向后”的barrier就等于将table分成了“未变”和“已变”两种状态。只要一个table改变了一次，就将其变成gray，等待重新扫描。被变成gray的table在被重新扫描之前，无论引用再发生多少次变化也都无关紧要了。

引用关系变化最频繁的要数thread对象了。thread通过stack引用其他对象，而stack作为运行期栈，在一直不停地被修改。如果要监控这些引用变化，肯定会造成执行效率严重下降。所以lua并没有在所有的stack引用变化处加入barrier，而是直接假设stack就是变化的。所以thread对象就算被扫描完成，也不会被设置成black，而是再次设置成gray，等待再次扫描。


### **Upvalue**


Upvalue对象在垃圾回收中的处理是比较特殊的。

对于open状态的upvalue，其v指向的是一个stack上的TValue，所以open upvalue与thread的关系非常紧密。引用到open upvalue的只可能是其从属的thread，以及lua closure。如果没有lua closure引用这个open upvalue，就算他一定被thread引用着，也已经没有实际的意义了，应该被回收掉。也就是说thread对open upvalue的引用完全是一个弱引用。所以Lua没有将open upvalue当作一个独立的可回收对象，而是将其清理工作交给从属的thread对象来完成。在mark过程中，open upvalue对象只使用white和gray两个状态，来代表是否被引用到。通过上面的引用关系可以看到，有可能引用open upvalue的对象只可能被lua closure引用到。所以一个gray的open upvalue就代表当前有lua closure正在引用他，而这个lua closure不一定在这个thread的stack上面。在清扫阶段，thread对象会遍历所有从属于自己的open upvalue。如果不是gray，就说明当前没有lua closure引用这个open upvalue了，可以被销毁。

当退出upvalue的语法域或者thread被销毁，open upvalue会被close。所有close upvalue与thread已经没有弱引用关系，会被转化为一个普通的可回收对象，和其他对象一样进行独立的垃圾回收。