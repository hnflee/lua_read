# Lua 学习笔记：协程

原文引用于：http://timothyqiu.com/archives/lua-note-coroutines/

协程（coroutine）并不是 Lua 独有的概念，如果让我用一句话概括，那么大概就是：一种能够在运行途中主动中断，并且能够从中断处恢复运行的特殊函数。（嗯，其实不是函数。）





### 举个最原始的栗子

---

下面给出一个最简单的 Lua 中 coroutine 的用法演示：

```lua
function greet()
    print "hello world"
end

co = coroutine.create(greet) -- 创建 coroutine

print(coroutine.status(co))  -- 输出 suspended
print(coroutine.resume(co))  -- 输出 hello world
                             -- 输出 true (resume 的返回值)
print(coroutine.status(co))  -- 输出 dead
print(coroutine.resume(co))  -- 输出 false    cannot resume dead coroutine (resume 的返回值)
print(type(co))              -- 输出 thread
```

协程在创建时，需要把协程体函数传递给创建函数 ```create()```。新创建的协程处于 suspended 状态，可以使用 ```resume``` 让其运行，全部执行完成后协程处于 dead 状态。如果尝试 ```resume``` 一个 dead 状态的，则可以从 ```resume``` 返回值上看出执行失败。另外你还可以注意到 Lua 中协程（coroutine）的变量类型其实叫做「thread」Orz...

乍一看可能感觉和线程没什么两样，但需要注意的是 ```resume()``` 只有在 ```greet()``` 以某种形式「返回」后才会返回（所以说协程像函数）。


### 函数执行的中断与再开


---

单从上面这个例子，我们似乎可以得出结论：协程果然就是某种坑爹的函数调用方式啊。然而，协程的真正魅力来自于 ```resume``` 和 ```yield``` 这对好基友之间的羁绊。 

**函数 coroutine.resume(co[, val1, ...])**

开始或恢复执行协程 `co`。

如果是开始执行，`val1` 及之后的值都作为参数传递给协程体函数；如果是恢复执行，`val1` 及之后的值都作为`yield()` 的返回值传递。

第一个返回值（还记得 Lua 可以返回多个值吗？）为表示执行成功与否的布尔值。如果成功，之后的返回值是 `yield` 的参数；如果失败，第二个返回值为失败的原因（Lua 的很多函数都采用这种错误处理方式）。

当然，如果是协程体函数执行完毕 `return` 而不是 `yield()`，那么 `resume()` 第一个返回值后跟着的就是其返回值。


**函数 coroutine.yield(...)**


中断协程的执行，使得开启该协程的 `coroutine.resume()` 返回。再度调用 `coroutine.resume()` 时，会从该 `yield()` 处恢复执行。

当然，`yield()` 的所有参数都会作为 `resume()` 第一个返回值后的返回值返回。

OK，总结一下：当 `co = coroutine.create(f`) 时，`yield()` 和` resume()` 的关系如下图：

![](http://timothyqiu.com/usr/uploads/2012/12/3025821892.png)


### How coroutine makes life easier



---

如果要求给某个怪写一个 AI：**先向右走 30 帧，然后只要玩家进入视野就往反方向逃 15 帧**。该怎么写？

**传统做法**
经典的纯状态机做法。

```lua
-- 每帧的逻辑
function Monster:frame()
    self:state_func()
    self.state_frame_count = self.state_frame_count + 1
end

-- 切换状态
function Monster:set_next_state(state)
    self.state_func = state
    self.state_frame_count = 0
end

-- 首先向右走 30 帧
function Monster:state_walk_1()
    local frame = self.state_frame_count
    self:walk(DIRECTION_RIGHT)
    if frame > 30 then
        self:set_next_state(state_wait_for_player)
    end
end

-- 等待玩家进入视野
function Monster:state_wait_for_player()
    if self:get_distance(player) < self.range then
        self.direction = -self:get_direction_to(player)
        self:set_next_state(state_walk_2)
    end
end

-- 向反方向走 15 帧
function Monster:state_walk_2()
    local frame = self.state_frame_count;
    self:walk(self.direction)
    if frame > 15 then
        self:set_next_state(state_wait_for_player)
    end
end
```

协程做法
```lua
-- 每帧的逻辑
function Monster:frame()
    -- 首先向右走 30 帧
    for i = 1, 30 do
        self:walk(DIRECTION_RIGHT)
        self:wait()
    end

    while true do
        -- 等待玩家进入视野
        while self:get_distance(player) >= self.range do
            self:wait()
        end

        -- 向反方向走 15 帧
        self.direction = -self:get_direction_to(player)
        for i = 1, 15 do
            self:walk(self.direction)
            self:wait()
        end
    end
end

-- 该帧结束
function Monster:wait()
    coroutine.yield()
end
```
额外说一句，从 `wait()` 函数可以看出，Lua 的协程并不要求一定要从协程体函数中调用 `yield()`，这是和 Python 的一个区别。