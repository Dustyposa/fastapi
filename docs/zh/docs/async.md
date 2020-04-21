# 并发和  async / await

关于*路径操作函数*的`async def`语法的细节以及一些关于异步代码，并发，并行的一些背景知识。

## 有点忙?

<abbr title="太长了; 不要阅读"><strong>TL;DR:</strong></abbr>

如果你在使用需要用 `await` 调用的第三库，像这样：

```Python
results = await some_library()
```

那么，用 `async def` 声明你的*路径操作函数*，像这样： 

```Python hl_lines="2"
@app.get('/')
async def read_results():
    results = await some_library()
    return results
```

!!! note

​	你只能在用 `async def` 定义的函数内部使用 `await`。

---

如果你正在使用和一些组件（数据库，API，文件系统，等等）交互的第三方库，它们还没有支持使用 `await`，（这是目前大部分数据库相关的库的情况），那么正常声明你的*路径操作函数*，只用 `def`，像这样：

```Python hl_lines="2"
@app.get('/')
def results():
    results = some_library()
    return results
```

---

如果你的应用（以某种方式）不需要和其他任何组件交互并等待其响应，那么用 `async def`。

---

如果你不清楚，用常规的`def`。

---

**注意**：你可以根据你的需要在*路径操作函数*中混合使用 `def` 和 `async`，并使用最佳选项定义每一个函数。 FastAPI 会用最合适的方式。

无论如何，在上面的任一情况中，FastAPI 都可以以异步的方式工作，而且速度非常快。

但是按照上述的步骤，它能够进行一些性能优化。

## 技术细节

较新的 Python 版本已经支持**“异步代码”**，使用带 **`async`  以及 `await`** 语法的**“协程”**。

让我们通过下面的章节来了解一下这些词语：

* **异步代码**
* **`async` 和 `await`**
* **协程**

## 异步代码

异步代码意味着该语言可以通过一种方式，告诉计算机/程序在代码中的一些位置，它必须等待*别的东西*在别的地方结束。让我们把*别的东西*叫做“慢文件”。

所以在这段时间内，计算机可以去做一些其他工作，到“慢文件”结束。

然后计算机/程序会每当在再次等待新任务返回或者完成了当前正在执行的所有的工作时，回到原来停下的地方并继续执行它会看在等待的任务中，有没有任何一个任务已经完成它要做的任务了。

再然后，它需要完成第一个任务（比如说，我们的“慢文件”），并继续执行任何与其相关的任务。

“等待别的东西”通常是指相对“慢”（和处理器及 RAM 内存的速度相比）的 <abbr title="Input and Output">I/O</abbr>  操作，比如等待：

* 来自客户端通过网络发送的数据
* 你的程序通过网络发送给客户端的数据
* 通过系统读取磁盘文件的内容给你的程序
* 你的程序给系统写到磁盘上的内容
* 一个远程 API 操作
* 要完成的数据库操作
* 一个返回结果的数据库查询
* 等等。

由于执行时间大多都是由等待 <abbr title="Input and Output">I/O</abbr>  操作消耗，所以也叫它们“ I/O 密集型”。

被叫做“异步的”是因为计算机/程序不需要和慢任务“同步”，等待任务结束的确切时间，当有时间时就能拿到任务结果并继续工作。

相反，作为一个“异步的”系统，任务一旦完成，任务可以排队稍作等待（几微秒），让计算机/程序完成它要完成的时间，之后回来获取任务结果，并继续处理它们。

对于“同步的”（与“异步的”相反）来说，它们通常也会使用术语“ sequential （顺序的）”，因为计算机/程序在切换到不同的任务之前都会按照顺序执行所有步骤，即使这些任步骤需要等待。


### 并发与汉堡包

上面描述的这种 **异步的** 代码思想有时也被叫做 **“并发”**。与 **”并行“** 不同。

**并发** 和 **并行** 都是指“不同的事或多或少同时发生”。

但是 *并发* 和 *并行* 之前的细节是完全不同的。

要想看清其中的区别，可以想象下面这个关于汉堡的故事：

### 并发的汉堡

你和你对象一起去买快餐，你站成一排，收银员从你前面的人那里接订单。

然后轮到你了，你为你的恋人和你点了两份非常美味的汉堡。

到你支付的时候。

收银员对在厨房里面的人说了一些什么，这样他知道他必须为你准备汉堡包（尽管他现在正在为之前的客户准备）。

收银员给了你的取餐号。

在等待的时候，你和你对象人找了一张桌子，坐着和你对象聊了很长时间的天（因为你的汉堡非常美味，需要一些时间去准备）。

当你和你对象坐在餐桌上时，你在等汉堡的同时，你可以用这段时间来欣赏你的对象是多么的好，多么的可爱以及聪明。

在等待以及和你对象聊天的同时，你时不时会看看柜台的取餐号码，看看是不是轮到你了。

在某个时候，终于轮到你了。你来到柜台，取到你的汉堡包并回到座位。

你和你的对象吃着汉堡，度过了一段愉快的时光。

---

想象一下你就是故事中的计算机/程序。

当你在排队的时候，你是闲着的，等着排队，并没有做很”有生产力“的事情。但是排队的速度很快，因为收银员只接受点餐，所以还行。

Then, when it's your turn, you do actual "productive" work, you process the menu, decide what you want, get your crush's choice, pay, check that you give the correct bill or card, check that you are charged correctly, check that the order has the correct items, etc.

But then, even though you still don't have your burgers, your work with the cashier is "on pause", because you have to wait for your burgers to be ready.

But as you go away from the counter and seat on the table with a number for your turn, you can switch your attention to your crush, and "work" on that. Then you are again doing something very "productive", as is flirting with your crush.

Then the cashier says "I'm finished with doing the burgers" by putting your number on the counter display, but you don't jump like crazy immediately when the displayed number changes to your turn number. You know no one will steal your burgers because you have the number of your turn, and they have theirs. 

So you wait for your crush to finish the story (finish the current work / task being processed), smile gently and say that you are going for the burgers.

Then you go to the counter, to the initial task that is now finished, pick the burgers, say thanks and take them to the table. That finishes that step / task of interaction with the counter. That in turn, creates a new task, of "eating burgers", but the previous one of "getting burgers" is finished.

### Parallel Burgers

You go with your crush to get parallel fast food.

You stand in line while several (let's say 8) cashiers take the orders from the people in front of you.

Everyone before you is waiting for their burgers to be ready before leaving the counter because each of the 8 cashiers goes himself and prepares the burger right away before getting the next order.

Then it's finally your turn, you place your order of 2 very fancy burgers for your crush and you.

You pay.

The cashier goes to the kitchen.

You wait, standing in front of the counter, so that no one else takes your burgers before you, as there are no numbers for turns.

As you and your crush are busy not letting anyone get in front of you and take your burgers whenever they arrive, you cannot pay attention to your crush.

This is "synchronous" work, you are "synchronized" with the cashier/cook. You have to wait and be there at the exact moment that the cashier/cook finishes the burgers and gives them to you, or otherwise, someone else might take them.

Then your cashier/cook finally comes back with your burgers, after a long time waiting there in front of the counter.

You take your burgers and go to the table with your crush.

You just eat them, and you are done.

There was not much talk or flirting as most of the time was spent waiting in front of the counter.

---

In this scenario of the parallel burgers, you are a computer / program with two processors (you and your crush), both waiting and dedicating their attention to be "waiting on the counter" for a long time.

The fast food store has 8 processors (cashiers/cooks). While the concurrent burgers store might have had only 2 (one cashier and one cook).

But still, the final experience is not the best.

---

This would be the parallel equivalent story for burgers.

For a more "real life" example of this, imagine a bank.

Up to recently, most of the banks had multiple cashiers and a big line.

All of the cashiers doing all the work with one client after the other.

And you have to wait in the line for a long time or you lose your turn.

You probably wouldn't want to take your crush with you to do errands at the bank.

### Burger Conclusion

In this scenario of "fast food burgers with your crush", as there is a lot of waiting, it makes a lot more sense to have a concurrent system.

This is the case for most of the web applications.

Many, many users, but your server is waiting for their not-so-good connection to send their requests.

And then waiting again for the responses to come back.

This "waiting" is measured in microseconds, but still, summing it all, it's a lot of waiting in the end.

That's why it makes a lot of sense to use asynchronous code for web APIs.

Most of the existing popular Python frameworks (including Flask and Django) were created before the new asynchronous features in Python existed. So, the ways they can be deployed support parallel execution and an older form of asynchronous execution that is not as powerful as the new capabilities.

Even though the main specification for asynchronous web Python (ASGI) was developed at Django, to add support for WebSockets.

That kind of asynchronicity is what made NodeJS popular (even though NodeJS is not parallel) and that's the strength of Go as a programing language.

And that's the same level of performance</a> you get with **FastAPI**.

And as you can have parallelism and asynchronicity at the same time, you get higher performance than most of the tested NodeJS frameworks and on par with Go, which is a compiled language closer to C <a href="https://www.techempower.com/benchmarks/#section=data-r17&hw=ph&test=query&l=zijmkf-1" class="external-link" target="_blank">(all thanks to Starlette)</a>.

### Is concurrency better than parallelism?

Nope! That's not the moral of the story.

Concurrency is different than parallelism. And it is better on **specific** scenarios that involve a lot of waiting. Because of that, it generally is a lot better than parallelism for web application development. But not for everything.

So, to balance that out, imagine the following short story:

> You have to clean a big, dirty house.

*Yep, that's the whole story*.

---

There's no waiting anywhere, just a lot of work to be done, on multiple places of the house.

You could have turns as in the burgers example, first the living room, then the kitchen, but as you are not waiting for anything, just cleaning and cleaning, the turns wouldn't affect anything.

It would take the same amount of time to finish with or without turns (concurrency) and you would have done the same amount of work.

But in this case, if you could bring the 8 ex-cashier/cooks/now-cleaners, and each one of them (plus you) could take a zone of the house to clean it, you could do all the work in **parallel**, with the extra help, and finish much sooner.

In this scenario, each one of the cleaners (including you) would be a processor, doing their part of the job. 

And as most of the execution time is taken by actual work (instead of waiting), and the work in a computer is done by a <abbr title="Central Processing Unit">CPU</abbr>, they call these problems "CPU bound".

---

Common examples of CPU bound operations are things that require complex math processing.

For example:

* **Audio** or **image processing**
* **Computer vision**: an image is composed of millions of pixels, each pixel has 3 values / colors, processing that normally requires computing something on those pixels, all at the same time)
* **Machine Learning**: it normally requires lots of "matrix" and "vector" multiplications. Think of a huge spreadsheet with numbers and multiplying all of them together at the same time.
* **Deep Learning**: this is a sub-field of Machine Learning, so, the same applies. It's just that there is not a single spreadsheet of numbers to multiply, but a huge set of them, and in many cases, you use a special processor to build and / or use those models.

### Concurrency + Parallelism: Web + Machine Learning

With **FastAPI** you can take the advantage of concurrency that is very common for web development (the same main attractive of NodeJS).

But you can also exploit the benefits of parallelism and multiprocessing (having multiple processes running in parallel) for **CPU bound** workloads like those in Machine Learning systems.

That, plus the simple fact that Python is the main language for **Data Science**, Machine Learning and especially Deep Learning, make FastAPI a very good match for Data Science / Machine Learning web APIs and applications (among many others).

To see how to achieve this parallelism in production see the section about [Deployment](deployment.md){.internal-link target=_blank}.

## `async` and `await`

Modern versions of python have a very intuitive way to define asynchronous code. This makes it look just like normal "sequential" code and do the "awaiting" for you at the right moments.

When there is an operation that will require waiting before giving the results and has support for these new Python features, you can code it like:

```Python
burgers = await get_burgers(2)
```

The key here is the `await`. It tells Python that it has to wait for `get_burgers(2)` to finish doing its thing before storing the results in `burgers`. With that, Python will know that it can go and do something else in the meanwhile (like receiving another request).

For `await` to work, it has to be inside a function that supports this asynchronicity. To do that, you just declare it with `async def`:

```Python hl_lines="1"
async def get_burgers(number: int):
    # Do some asynchronous stuff to create the burgers
    return burgers
```

...instead of `def`:

```Python hl_lines="2"
# This is not asynchronous
def get_sequential_burgers(number: int):
    # Do some sequential stuff to create the burgers
    return burgers
```

With `async def`, Python knows that, inside that function, it has to be aware of `await` expressions, and that it can "pause" the execution of that function and go do something else before coming back.

When you want to call an `async def` function, you have to "await" it. So, this won't work:

```Python
# This won't work, because get_burgers was defined with: async def
burgers = get_burgers(2)
```

---

So, if you are using a library that tells you that you can call it with `await`, you need to create the *path operation functions* that uses it with `async def`, like in:

```Python hl_lines="2 3"
@app.get('/burgers')
async def read_burgers():
    burgers = await get_burgers(2)
    return burgers
```

### More technical details

You might have noticed that `await` can only be used inside of functions defined with `async def`.

But at the same time, functions defined with `async def` have to be "awaited". So, functions with `async def` can only be called inside of functions defined with `async def` too.

So, about the egg and the chicken, how do you call the first `async` function?

If you are working with **FastAPI** you don't have to worry about that, because that "first" function will be your *path operation function*, and FastAPI will know how to do the right thing.

But if you want to use `async` / `await` without FastAPI, <a href="https://docs.python.org/3/library/asyncio-task.html#coroutine" class="external-link" target="_blank">check the official Python docs</a>.

### Other forms of asynchronous code

This style of using `async` and `await` is relatively new in the language.

But it makes working with asynchronous code a lot easier.

This same syntax (or almost identical) was also included recently in modern versions of JavaScript (in Browser and NodeJS).

But before that, handling asynchronous code was quite more complex and difficult.

In previous versions of Python, you could have used threads or <a href="http://www.gevent.org/" class="external-link" target="_blank">Gevent</a>. But the code is way more complex to understand, debug, and think about.

In previous versions of NodeJS / Browser JavaScript, you would have used "callbacks". Which lead to <a href="http://callbackhell.com/" class="external-link" target="_blank">callback hell</a>.

## Coroutines

**Coroutine** is just the very fancy term for the thing returned by an `async def` function. Python knows that it is something like a function that it can start and that it will end at some point, but that it might be paused internally too, whenever there is an `await` inside of it.

But all this functionality of using asynchronous code with `async` and `await` is many times summarized as using "coroutines". It is comparable to the main key feature of Go, the "Goroutines".

## Conclusion

Let's see the same phrase from above:

> Modern versions of Python have support for **"asynchronous code"** using something called **"coroutines"**, with **`async` and `await`** syntax.

That should make more sense now.

All that is what powers FastAPI (through Starlette) and what makes it have such an impressive performance.

## Very Technical Details

!!! warning
    You can probably skip this.
    
    These are very technical details of how **FastAPI** works underneath.
    
    If you have quite some technical knowledge (co-routines, threads, blocking, etc) and are curious about how FastAPI handles `async def` vs normal `def`, go ahead.

### Path operation functions

When you declare a *path operation function* with normal `def` instead of `async def`, it is run in an external threadpool that is then awaited, instead of being called directly (as it would block the server).

If you are coming from another async framework that does not work in the way described above and you are used to define trivial compute-only *path operation functions* with plain `def` for a tiny performance gain (about 100 nanoseconds), please note that in **FastAPI** the effect would be quite opposite. In these cases, it's better to use `async def` unless your *path operation functions* use code that performs blocking <abbr title="Input/Output: disk reading or writing, network communications.">IO</abbr>.

Still, in both situations, chances are that **FastAPI** will [still be faster](/#performance){.internal-link target=_blank} than (or at least comparable to) your previous framework.

### Dependencies

The same applies for dependencies. If a dependency is a standard `def` function instead of `async def`, it is run in the external threadpool.

### Sub-dependencies

You can have multiple dependencies and sub-dependencies requiring each other (as parameters of the function definitions), some of them might be created with `async def` and some with normal `def`. It would still work, and the ones created with normal `def` would be called on an external thread instead of being "awaited".

### Other utility functions

Any other utility function that you call directly can be created with normal `def` or `async def` and FastAPI won't affect the way you call it.

This is in contrast to the functions that FastAPI calls for you: *path operation functions* and dependencies.

If your utility function is a normal function with `def`, it will be called directly (as you write it in your code), not in a threadpool, if the function is created with `async def` then you should await for that function when you call it in your code.

---

Again, these are very technical details that would probably be useful if you came searching for them.

Otherwise, you should be good with the guidelines from the section above: <a href="#in-a-hurry">In a hurry?</a>.
