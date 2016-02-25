# Lua的协程和协程库详解



我们首先介绍一下什么是协程、然后详细介绍一下coroutine库，然后介绍一下协程的简单用法，最后介绍一下协程的复杂用法。

一、协程是什么？

（1）线程

首先复习一下多线程。我们都知道线程——Thread。每一个线程都代表一个执行序列。

当我们在程序中创建多线程的时候，看起来，同一时刻多个线程是同时执行的，不过实质上多个线程是并发的，因为只有一个CPU，所以实质上同一个时刻只有一个线程在执行。

在一个时间片内执行哪个线程是不确定的，我们可以控制线程的优先级，不过真正的线程调度由CPU的调度决定。

（2）协程

那什么是协程呢？协程跟线程都代表一个执行序列。不同的是，协程把线程中不确定的地方尽可能的去掉，执行序列间的切换不再由CPU隐藏的进行，而是由程序显式的进行。

所以，使用协程实现并发，需要多个协程彼此协作。
二、resume和yeild的协作。

resume和yeild的协作是Lua协程的核心。这边用一幅图描述一下，有一个大体的印象。对照下面的coroutine库的详细解释和最后的代码，应该可以搞清楚协程的概念了。

注：这是在非首次resume协程的情况下，resume和yield的互相调用的情况。如果是首次resume协程，那么resume的参数会直接传递给协程函数。

![](http://img1.tuicool.com/AjaEzi.png!web)

三、coroutine库详解

(1)coroutine.create (f)

传一个函数参数，用来创建协程。返回一个“thread”对象。

(2)coroutine.isyieldable ()

如果正在运行的协程可以让出，则返回真。值得注意的是，只有主协程（线程）和C函数中是无法让出的。

(3)coroutine.resume (co [, val1, ···])

这是一个非常重要的函数。用来启动或再次启动一个协程，使其由挂起状态变成运行状态。

可以这么说，resume函数相当于在执行协程中的方法。参数Val1...是执行协程co时传递给协程的方法。
首次执行协程co时，参数Val1...会传递给协程co的函数；
再次执行协程co时，参数Val1...会作为给协程co中上一次yeild的返回值。

不知道这句话大家理解了没，这是协程的核心。如果没理解也不用急，继续往下看，稍后我会详细解释。

resume函数返回什么呢？有3种情况：

1）、如果协程co的函数执行完毕，协程正常终止， resume 返回 true和函数的返回值。

2）、如果协程co的函数执行过程中，协程让出了（调用了yeild()方法），那么resume返回true和协程中调用yeild传入的参数。

3）、如果协程co的函数执行过程中发生错误，resume返回false与错误消息。

可以看到resume无论如何都不会导致程序崩溃。它是在保护模式下执行的。

(4)coroutine.running ()

用来判断当前执行的协程是不是主线程，如果是，就返回true。

(5)coroutine.status (co)

返回一个字符串，表示协程的状态。有4种状态：

1）、running。如果在协程的函数中调用status，传入协程自身的句柄，那么执行到这里的时候才会返回running状态。

2）、suspended。如果协程还未结束，即自身调用了yeild或还没开始运行，那么就是suspended状态。

3）、normal。如果协程Aresume协程B时，协程A处于的状态为normal。在协程B的执行过程中，协程A就一直处于normal状态。因为它这时候既不是挂起状态、也不是运行状态。

4）、dead。如果一个协程发生错误结束，或正常终止。那么就处于dead状态。如果这时候对它调用resume，将返回false和错误消息。

(6)coroutine.wrap (f)

wrap()也是用来创建协程的。只不过这个协程的句柄是隐藏的。跟create()的区别在于：

1）、wrap()返回的是一个函数，每次调用这个函数相当于调用coroutine.resume()。

2）、调用这个函数相当于在执行resume()函数。

3）、调用这个函数时传入的参数，就相当于在调用resume时传入的除协程的句柄外的其他参数。

4）、调用这个函数时，跟resume不同的是，它并不是在保护模式下执行的，若执行崩溃会直接向外抛出。

(7)coroutine.yield (···)

使正在执行的函数挂起。
传递给yeild的参数会作为resume的额外返回值。

同时，如果对该协程不是第一次执行resume，resume函数传入的参数将会作为yield的返回值。
四、例子进阶。

（1）、例子1：简单实用resume、yield，如下：
```lua
coco = coroutine.create(function (a,b)
  print("resume args:"..a..","..b)
  yreturn = coroutine.yield()
  print ("yreturn :"..yreturn)
end)
coroutine.resume(coco,0,1)
coroutine.resume(coco,21)
```

输出：
```
resume args:0,1
yreturn :21
```

（2）、例子2：简单使用wrap，如下：
```lua
coco2 = coroutine.wrap(function (a,b)
  print("resume args:"..a..","..b)
  yreturn = coroutine.yield()
  print ("yreturn :"..yreturn)
end)
print(type(coco2))
coco2(0,1)
coco2(21)
```
输出：
```
function
resume args:0,1
yreturn :21
```
很明显，wrap的使用更方便。

（3）、如果还没有足够的理解，且看我放大招，看这个例子：
```lua
function status()
  print("co1's status :"..coroutine.status(co1).." ,co2's status: "..coroutine.status(co2))
end
co1 = coroutine.create(function ( a )
  print("arg is :"..a)
  status()
  local stat,rere = coroutine.resume(co2,"2")
  print("resume's return is "..rere)
  status()
  local stat2,rere2 = coroutine.resume(co2,"4")
  print("resume's return is "..rere2)
  local arg = coroutine.yield("6")
end)
co2 = coroutine.create(function ( a )
  print("arg is :"..a)
  status()
  local rey = coroutine.yield("3")
  print("yeild's return is " .. rey)
  status()
  coroutine.yield("5")
end)
--主线程执行co1,传入字符串“main thread arg”
stat,mainre = coroutine.resume(co1,"1")
status()
print("last return is "..mainre)
```
用一个函数status()输出2个协程的状态,最后输出如下：
```
arg is :1
co1's status :running ,co2's status: suspended
arg is :2
co1's status :normal ,co2's status: running
resume's return is 3
co1's status :running ,co2's status: suspended
yeild's return is 4
co1's status :normal ,co2's status: running
resume's return is 5
co1's status :suspended ,co2's status: suspended
last return is 6
```
（4）、最后附一个云风在Lua5.3参考手册中给出的例子：
```lua
function foo(a)
  print("foo", a)
  return coroutine.yield(2 * a)
end
co = coroutine.create(function ( a, b )
  print("co-body", a, b)
  local r = foo(a + 1)
  print("co-body", r)
  local r, s = coroutine.yield(a + b, a - b)
  print("co-body", r, s)
  return b, "end"
end)
print("main", coroutine.resume(co, 1, 10))
print("main", coroutine.resume(co, "r"))
print("main", coroutine.resume(co, "x", "y"))
print("main", coroutine.resume(co, "x", "y"))
```
输出如下：
```
co-body    1    10
foo    2
main    true    4
co-body    r
main    true    11    -9
co-body    x    y
main    true    10    end
main    false    cannot resume dead coroutine

```