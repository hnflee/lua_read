# 内部实现:Garbage Collection(1) 原理

Lua5.2采用垃圾回收机制对所有的lua对象(GCObject)进行管理。Lua虚拟机会定期运行GC，释放掉已经不再被被引用到的lua对象。


### **基本算法**

基本的垃圾回收算法被称为"mark-and-sweep"算法。算法本身其实很简单。

首先，系统管理着所有已经创建了的对象。每个对象都有对其他对象的引用。root集合代表着已知的系统级别的对象引用。我们从root集合出发，就可以访问到系统引用到的所有对象。而没有被访问到的对象就是垃圾对象，需要被销毁。

我们可以将所有对象分成三个状态：

1. White状态，也就是待访问状态。表示对象还没有被垃圾回收的标记过程访问到。
2. Gray状态，也就是待扫描状态。表示对象已经被垃圾回收访问到了，但是对象本身对于其他对象的引用还没有进行遍历访问。
3. Black状态，也就是已扫描状态。表示对象已经被访问到了，并且也已经遍历了对象本身对其他对象的引用。


基本的算法可以描述如下：

```
    当前所有对象都是White状态;  
    将root集合引用到的对象从White设置成Gray，并放到Gray集合中;  
    while(Gray集合不为空)  
    {  
        从Gray集合中移除一个对象O，并将O设置成Black状态;  
        for(O中每一个引用到的对象O1) {  
            if(O1在White状态) {  
                将O1从White设置成Gray，并放到到Gray集合中；  
            }  
        }  
    }  
    for(任意一个对象O){  
        if(O在White状态)  
            销毁对象O;  
        else  
            将O设置成White状态;  
    } 
```

###**Incremental Garbage Collection**
 
上面的算法如果一次性执行，在对象很多的情况下，会执行很长时间，严重影响程序本身的响应速度。其中一个解决办法就是，可以将上面的算法分步执行，这样每个步骤所耗费的时间就比较小了。我们可以将上述算法改为以下下几个步骤。

首先标识所有的root对象：

```
    当前所有对象都是White状态;  
    将root集合引用到的对象从White设置成Gray，并放到Gray集合中;
```

遍历访问所有的gray对象。如果超出了本次计算量上限，退出等待下一次遍历:


```
    while(Gray集合不为空,并且没有超过本次计算量的上限){  
            从Gray集合中移除一个对象O，并将O设置成Black状态;  
            for(O中每一个引用到的对象O1) {  
                if(O1在White状态) {  
                    将O1从White设置成Gray，并放到到Gray集合中；  
                }  
            }  
        }  
```

销毁垃圾对象：

```
    for(任意一个对象O){  
        if(O在White状态)  
            销毁对象O;  
        else  
            将O设置成White状态;  
    }  
```

在每个步骤之间，由于程序可以正常执行，所以会破坏当前对象之间的引用关系。black对象表示已经被扫描的对象，所以他应该不可能引用到一个white对象。当程序的改变使得一个black对象引用到一个white对象时，就会造成错误。解决这个问题的办法就是设置barrier。barrier在程序正常运行过程中，监控所有的引用改变。如果一个black对象需要引用一个white对象，存在两种处理办法：

1.     将white对象设置成gray，并添加到gray列表中等待扫描。这样等于帮助整个GC的标识过程向前推进了一步。
2.     将black对象该回成gray，并添加到gray列表中等待扫描。这样等于使整个GC的标识过程后退了一步。

这种垃圾回收方式被称为"Incremental Garbage Collection"(简称为"IGC"，Lua所采用的就是这种方法。使用"IGC"并不是没有代价的。IGC所检测出来的垃圾对象集合比实际的集合要小，也就是说，有些在GC过程中变成垃圾的对象，有可能在本轮GC中检测不到。不过，这些残余的垃圾对象一定会在下一轮GC被检测出来，不会造成泄露