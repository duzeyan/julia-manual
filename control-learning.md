# 控制流

Julia 提供一系列控制流：

* [复合表达式](##复合表达式) ： `begin` 和 `(;)`
* [条件求值](##条件求值) ： `if-elseif-else` 和 `?: (ternary operator)`
* [短路求值](##短路求值) ： `&&, ||` 和 `chained comparisons`
* [重复求值: 循环](##重复求值:循环) ： `while` 和 `for`
* [异常处理](##异常处理) ： `try-catch` ， `error` 和 `throw`
* [任务（也称为协程）](##任务（也称为协程）) ： `yieldto`

前五个控制流机制是高级编程语言的标准。但任务不是：它提供了非本地的控制流，便于在临时暂停的计算中进行切换。在 Julia 中，异常处理和协同多任务都是使用的这个机制。

## 复合表达式

用一个表达式按照顺序对一系列子表达式求值，并返回最后一个子表达式的值，有两种方法： `begin` 块和 `(;)` 链。 `begin` 块的例子：

```
julia> z = begin
         x = 1
         y = 2
         x + y
       end
3
```

这个块很短也很简单，可以用 (;) 链语法将其放在一行上：

```
julia> z = (x = 1; y = 2; x + y)
3
```

这个语法在[函数]()中的单行函数定义非常有用。 begin 块也可以写成单行， (;) 链也可以写成多行：

```
julia> begin x = 1; y = 2; x + y end
3

julia> (x = 1;
        y = 2;
        x + y)
3
```

## 条件求值

一个 `if-elseif-else` 条件表达式的例子：

```
if x < y
  println("x is less than y")
elseif x > y
  println("x is greater than y")
else
  println("x is equal to y")
end
```

如果条件表达式 `x < y` 为真，相应的语句块将会被执行；否则就执行条件表达式 `x > y` ，如果结果为真, 相应的语句块将被执行；如果两个表达式都是假， `else` 语句块将被执行。这是它用在实际中的例子：

```
julia> function test(x, y)
         if x < y
           println("x is less than y")
         elseif x > y
           println("x is greater than y")
         else
           println("x is equal to y")
         end
       end
test (generic function with 1 method)

julia> test(1, 2)
x is less than y

julia> test(2, 1)
x is greater than y

julia> test(1, 1)
x is equal to y
```

`elseif` 及 `else` 块是可选的。

Note that very short conditional statements (one-liners) are frequently expressed using Short-Circuit Evaluation in Julia, as outlined in the next section.

如果条件表达式的值是除 `true` 和 `false` 之外的值，会出错：

```
julia> if 1
         println("true")
       end
ERROR: type: non-boolean (Int64) used in boolean context
```

“问号表达式”语法 `?:` 与 `if-elseif-else` 语法相关，但是适用于单个表达式：

```
a ? b : c
```

`?` 之前的 `a` 是条件表达式，如果为 `true` ，就执行 : 之前的 `b` 表达式，如果为 `false` ，就执行 `:` 的 `c` 表达式。

用问号表达式来重写，可以使前面的例子更加紧凑。先看一个二选一的例子：

```
julia> x = 1; y = 2;

julia> println(x < y ? "less than" : "not less than")
less than

julia> x = 1; y = 0;

julia> println(x < y ? "less than" : "not less than")
not less than
```

三选一的例子需要链式调用问号表达式：

```
julia> test(x, y) = println(x < y ? "x is less than y"    :
                            x > y ? "x is greater than y" : "x is equal to y")
test (generic function with 1 method)

julia> test(1, 2)
x is less than y

julia> test(2, 1)
x is greater than y

julia> test(1, 1)
x is equal to y
```

链式问号表达式的结合规则是从右到左。

与 `if-elseif-else` 类似， : 前后的表达式，只有在对应条件表达式为 `true` 或 `false` 时才执行：

```
julia> v(x) = (println(x); x)
v (generic function with 1 method)


julia> 1 < 2 ? v("yes") : v("no")
yes
"yes"

julia> 1 > 2 ? v("yes") : v("no")
no
"no"
```

## 短路求值

`&&` 和 `||` 布尔运算符被称为短路求值，它们连接一系列布尔表达式，仅计算最少的表达式来确定整个链的布尔值。这意味着：
在表达式 `a && b` 中，只有 `a` 为 `true` 时才计算子表达式 `b`
在表达式 `a || b` 中，只有 `a` 为 `false` 时才计算子表达式 `b`
`&&` 和 `||` 都与右侧结合，但 `&&` 比 `||` 优先级高：

```
julia> t(x) = (println(x); true)
t (generic function with 1 method)

julia> f(x) = (println(x); false)
f (generic function with 1 method)

julia> t(1) && t(2)
1
2
true

julia> t(1) && f(2)
1
2
false

julia> f(1) && t(2)
1
false

julia> f(1) && f(2)
1
false

julia> t(1) || t(2)
1
true

julia> t(1) || f(2)
1
true

julia> f(1) || t(2)
1
2
true

julia> f(1) || f(2)
1
2
false
```

这种方式在 Julia 里经常作为 if 语句的一个简洁的替代。 可以把 `if <cond> <statement> end` 写成 `<cond> && <statement> (读作 <cond> *从而* <statement>)`。 类似地， 可以把 `if ! <cond> <statement> end 写成 `<cond> || <statement> (读作 <cond> 要不就 <statement>)`。

例如, 递归阶乘可以这样写:

```
julia> function factorial(n::Int)
           n >= 0 || error("n must be non-negative")
           n == 0 && return 1
           n * factorial(n-1)
       end
factorial (generic function with 1 method)

julia> factorial(5)
120

julia> factorial(0)
1

julia> factorial(-1)
ERROR: n must be non-negative
 in factorial at none:2
```

非 短路求值运算符，可以使用[数学运算和基本函数](operation-function.md)中介绍的位布尔运算符 `&` 和 `|` ：

```
julia> f(1) & t(2)
1
2
false

julia> t(1) | t(2)
1
2
true
```

`&&` 和 `||` 的运算对象也必须是布尔值（ `true` 或 `false` ）。Using a non-boolean value anywhere except for the last entry in a conditional chain is an error ：

```
julia> 1 && true
ERROR: type: non-boolean (Int64) used in boolean context
```

On the other hand, any type of expression can be used at the end of a conditional chain. It will be evaluated and returned depending on the preceding conditionals:

```
julia> true && (x = rand(2,2))
2x2 Array{Float64,2}:
 0.768448  0.673959
 0.940515  0.395453

julia> false && (x = rand(2,2))
false
```

## 重复求值: 循环

有两种循环表达式： `while` 循环和 `for` 循环。下面是 `while` 的例子：

```
julia> i = 1;

julia> while i <= 5
         println(i)
         i += 1
       end
1
2
3
4
5
```

上例也可以重写为 `for` 循环：

```
julia> for i = 1:5
         println(i)
       end
1
2
3
4
5
```

此处的 `1:5` 是一个 `Range` 对象，表示的是 1, 2, 3, 4, 5 序列。 `for` 循环遍历这些数，将其逐一赋给变量 `i` 。 `while` 循环和 `for` 循环的另一区别是变量的作用域。如果在其它作用域中没有引入变量 `i` ，那么它仅存在于 `for` 循环中。不难验证：

```
julia> for j = 1:5
         println(j)
       end
1
2
3
4
5

julia> j
ERROR: j not defined
```

有关变量作用域，详见[变量的作用域](applied-force.md) 。

通常， `for` 循环可以遍历任意容器。这时，应使用另一个（但是完全等价的）关键词 `in` ，而不是 `=` ，它使得代码更易阅读：

```
julia> for i in [1,4,0]
         println(i)
       end
1
4
0

julia> for s in ["foo","bar","baz"]
         println(s)
       end
foo
bar
baz
```

手册中将介绍各种可迭代容器（详见[多维数组](mutil-array.md)）。

有时要提前终止 `while` 或 `for` 循环。可以通过关键词 `break` 来实现：

```
julia> i = 1;

julia> while true
         println(i)
         if i >= 5
           break
         end
         i += 1
       end
1
2
3
4
5

julia> for i = 1:1000
         println(i)
         if i >= 5
           break
         end
       end
1
2
3
4
5
```

有时需要中断本次循环，进行下一次循环，这时可以用关键字 `continue` ：

```
julia> for i = 1:10
         if i % 3 != 0
           continue
         end
         println(i)
       end
3
6
9
```

多层 `for` 循环可以被重写为一个外层循环，迭代类似于笛卡尔乘积的形式：

```
julia> for i = 1:2, j = 3:4
         println((i, j))
       end
(1,3)
(1,4)
(2,3)
(2,4)
```

这种情况下用 `break` 可以直接跳出所有循环。

## 异常处理

当遇到意外条件时，函数可能无法给调用者返回一个合理值。这时，要么终止程序，打印诊断错误信息；要么程序员编写异常处理。

### 内置异常 `Exception`

如果程序遇到意外条件，异常将会被抛出。表中列出内置异常。

|Exception|
|:---|
|ArgumentError|
|BoundsError|
|DivideError|
|DomainError|
|EOFError|
|ErrorException|
|InexactError|
|InterruptException|
|KeyError|
|LoadError|
|MemoryError|
|MethodError|
|OverflowError|
|ParseError|
|SystemError|
|TypeError|
|UndefRefError|
|UndefVarError|
例如，当对负实数使用内置的 `sqrt` 函数时，将抛出 `DomainError()` ：

```
julia> sqrt(-1)
ERROR: DomainError
sqrt will only return a complex result if called with a complex argument.
try sqrt(complex(x))
 in sqrt at math.jl:131
```

You may define your own exceptions in the following way:

```
julia> type MyCustomException <: Exception end
```

### `throw` 函数

可以使用 `throw` 函数显式创建异常。例如，某个函数只对非负数做了定义，如果参数为负数，可以抛出 `DomaineError` 异常：

```
julia> f(x) = x>=0 ? exp(-x) : throw(DomainError())
f (generic function with 1 method)

julia> f(1)
0.36787944117144233

julia> f(-1)
ERROR: DomainError
 in f at none:1
```

注意， `DomainError` 使用时需要使用带括号的形式，否则返回的并不是异常，而是异常的类型。必须带括号才能返回 `Exception` 对象：

```
julia> typeof(DomainError()) <: Exception
true

julia> typeof(DomainError) <: Exception
false
Additionally, some exception types take one or more arguments that are used for error reporting:

julia> throw(UndefVarError(:x))
ERROR: x not defined
This mechanism can be implemented easily by custom exception types following the way UndefVarError is written:

julia> type MyUndefVarError <: Exception
           var::Symbol
       end
julia> Base.showerror(io::IO, e::MyUndefVarError) = print(io, e.var, " not defined");
```

### `error` 函数

`error` 函数用来产生 `ErrorException` ，阻断程序的正常执行。

如下改写 `sqrt` 函数，当参数为负数时，提示错误，立即停止执行：

```
julia> fussy_sqrt(x) = x >= 0 ? sqrt(x) : error("negative x not allowed")
fussy_sqrt (generic function with 1 method)

julia> fussy_sqrt(2)
1.4142135623730951

julia> fussy_sqrt(-1)
ERROR: negative x not allowed
 in fussy_sqrt at none:1
```

当对负数调用 `fussy_sqrt` 时，它会立即返回，显示错误信息：

```
julia> function verbose_fussy_sqrt(x)
         println("before fussy_sqrt")
         r = fussy_sqrt(x)
         println("after fussy_sqrt")
         return r
       end
verbose_fussy_sqrt (generic function with 1 method)

julia> verbose_fussy_sqrt(2)
before fussy_sqrt
after fussy_sqrt
1.4142135623730951

julia> verbose_fussy_sqrt(-1)
before fussy_sqrt
ERROR: negative x not allowed
 in verbose_fussy_sqrt at none:3
```

### `warn` 和 `info` 函数

Julia 还提供一些函数，用来向标准错误 I/O 输出一些消息，但不抛出异常，因而并不会打断程序的执行：

```
julia> info("Hi"); 1+1
INFO: Hi
2

julia> warn("Hi"); 1+1
WARNING: Hi
2

julia> error("Hi"); 1+1
ERROR: Hi
 in error at error.jl:21
```

### `try/catch` 语句

`try/catch` 语句可以用于处理一部分预料中的异常 `Exception` 。例如，下面求平方根函数可以正确处理实数或者复数：

```
julia> f(x) = try
         sqrt(x)
       catch
         sqrt(complex(x, 0))
       end
f (generic function with 1 method)

julia> f(1)
1.0

julia> f(-1)
0.0 + 1.0im
```

但是处理异常比正常采用分支来处理，会慢得多。

`try/catch` 语句使用时也可以把异常赋值给某个变量。例如：

```
julia> sqrt_second(x) = try
         sqrt(x[2])
       catch y
         if isa(y, DomainError)
           sqrt(complex(x[2], 0))
         elseif isa(y, BoundsError)
           sqrt(x)
         end
       end
sqrt_second (generic function with 1 method)

julia> sqrt_second([1 4])
2.0

julia> sqrt_second([1 -4])
0.0 + 2.0im

julia> sqrt_second(9)
3.0

julia> sqrt_second(-9)
ERROR: DomainError
 in sqrt_second at none:7
```

Note that the symbol following catch will always be interpreted as a name for the exception, so care is needed when writing try/catch expressions on a single line. The following code will not work to return the value of x in case of an error:

```
try bad() catch x end
```

Instead, use a semicolon or insert a line break after catch:

```
try bad() catch; x end

try bad()
catch
  x
end
```

Julia 还提供了更高级的异常处理函数 `rethrow` ， `backtrace` 和 `catch_backtrace` 。

### finally 语句

在改变状态或者使用文件等资源时，通常需要在操作执行完成时做清理工作（比如关闭文件）。异常的存在使得这样的任务变得复杂，因为异常会导致程序提前退出。关键字 finally 可以解决这样的问题，无论程序是怎样退出的， finally 语句总是会被执行。

例如, 下面的程序说明了怎样保证打开的文件总是会被关闭：

```
f = open("file")
try
    # operate on file f
finally
    close(f)
end
```

当程序执行完 `try` 语句块（例如因为执行到 `return` 语句，或者只是正常完成）， `close` 语句将会被执行。如果 `try` 语句块因为异常提前退出，异常将会继续传播。 `catch` 语句可以和 `try` ， `finally` 一起使用。这时。 `finally` 语句将会在 `catch` 处理完异常之后执行。


## 任务（也称为协程）

任务是一种允许计算灵活地挂起和恢复的控制流，有时也被称为对称协程、轻量级线程、协同多任务等。

如果一个计算（比如运行一个函数）被设计为 `Task` ，有可能因为切换到其它 `Task` 而被中断。原先的 `Task` 在以后恢复时，会从原先中断的地方继续工作。切换任务不需要任何空间，同时可以有任意数量的任务切换，而不需要考虑堆栈问题。任务切换与函数调用不同，可以按照任何顺序来进行。

任务比较适合生产者-消费者模式，一个过程用来生产值，另一个用来消费值。消费者不能简单的调用生产者来得到值，因为两者的执行时间不一定协同。在任务中，两者则可以 正常运行。

Julia 提供了 `produce` 和 `consume` 函数来解决这个问题。生产者调用 `produce` 函数来生产值：

```
julia> function producer()
         produce("start")
         for n=1:4
           produce(2n)
         end
         produce("stop")
       end;
```

要消费生产的值，先对生产者调用 `Task` 函数，然后对返回的对象重复调用 `consume` ：

```
julia> p = Task(producer);

julia> consume(p)
"start"

julia> consume(p)
2

julia> consume(p)
4

julia> consume(p)
6

julia> consume(p)
8

julia> consume(p)
"stop"
```

可以在 for 循环中迭代任务，生产的值被赋值给循环变量：

```
julia> for x in Task(producer)
         println(x)
       end
start
2
4
6
8
stop
```

注意 `Task()` 函数的参数，应为零参函数。生产者常常是参数化的，因此需要为其构造零参[匿名函数](function-learning.md) 。可以直接写，也可以调用宏：

```
function mytask(myarg)
    ...
end

taskHdl = Task(() -> mytask(7))
# 也可以写成
taskHdl = @task mytask(7)
```

`produce` 和 `consume` 但它并不在不同的 CPU 发起线程。我们将在 [并行计算](concurrent-computation.md)中，讨论真正的内核线程。

### 核心任务操作

尽管 `produce` 和 `consume` 已经阐释了任务的本质，但是他们实际上是由库函数调用更原始的函数 `yieldt`o 实现的。 `yieldto(task,value)` 挂起当前任务，切换到特定的 `tas`k ， 并使这个 `task` 的最后一次 `yeidlto` 返回 特定的 `value`。注意 `yieldto` 是唯一需要的操作来进行 ‘任务风格’的控制流; 不需要调用和返回，我们只用在不同的任务之间切换即可。 这就是为什么这个特性被称做 “对称式协程”;每一个任务的切换都是用相同的机制。

`yeildto` 很强大， 但是大多数时候并不直接调用它。 当你从当前的任务切换走，你有可能会想切换回来， 但需要知道切换的时机和任务，这会需要相当的协调。 例如， `procude` 需要保持某个状态来记录消费者。 无需手动地记录正在消费的任务让 `produce` 比 `yieldto` 更容易使用。

除此之外， 为了高效地使用任务，其他一些基本的函数也同样必须。 `current_task()` 获得当前运行任务的引用。 `istaskdone(t)` 查询任务是否终止。 istaskstarted(t) 查询任务是否启动。 `task_local_storage` 处理当前任务的键值储存。

### 任务与事件

大多数任务的切换都是在等待像 I/O 请求这样的事件的时候，并由标准库的调度器完成。调度器记录正在运行 的任务的队列，并执行一个循环来根据外部事件(比如消息到达)重启任务。

处理等待事件的基本函数是 `wait`。 有几种对象实现了 `wait`，比如对于 `Process` 对象， `wait` 会等待它终止。更多的时候 `wait` 是隐式的， 比如 `wait` 可以发生在调用 `read` 的时候，等待数据变得可用。

在所有的情况中, `wait` 最终会操作在一个负责将任务排队和重启的 `Condition` 对象上。 当任务在 `Condition` 上调用 `wait`， 任务会被标记为不可运行，被加入到 `Condition` 的 队列中，再切换至调度器。 调度器会选取另一个任务来运行， 或者等待外部事件。 如果一切正常， 最终一个事件句柄 会在 `Condition` 上调用 `notif`y， 使正在等待的任务变得可以运行。

调用 `Task` 可以生成一个初始对调度器还未知的任务， 这允许你用 `yieldto` 手动管理任务。不管怎样， 当这样的任务正在等待事件时，事件一旦发生，它仍然会自动重启。而且任何时候你都可以 调用 `schedule(task)` 或者用宏 `@schedule` 或 `@async` 来让调度器来运行一个任务， 根本不用去等待任何事件。(参见[并行计算](concurrent-computation.md))

### 任务状态

任务包含一个 `state` 域，它用来描述任务的执行状态。任务状态取如下的几种符号中的一种：

|符号	|意义|
|:-----|:------|
|:runnable	|任务正在运行，或可被切换到该任务|
|:waiting	|Blocked waiting for a specific event|
|:queued|	In the scheduler’s run queue about to be restarted|
|:done	|成功执行完毕|
|:failed	|由于未处理的异常而终止|
