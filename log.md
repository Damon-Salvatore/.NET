## 使用 log4net 作为日志框架时，如何正确记录日志

首先：日志按等级划分，`log4net`里分为了为 `Error`、`Info`、 `Debug`、 `Warn`四种，

分级可以明确事件的重要程度。

### 针对 Error Log

`error log` 相较于 `info log` 更具体，更有方法论，所以我们先说 `error log`

在我们针对 `error log` 怎么正确记录之前，我们先看如何正确抛出异常

有的程序员经常在开发中编写这样的代码:

```cs
try
{
  // do stuff that might throw an exception
}
catch(Exception e)
{
  throw e;
}
```

上面的代码有两个问题:

1. `throw ex` 将异常中的调用堆栈重置到抛出语句所在的位置;丢失有关异常实际创建位置的信息。

2. 只是捕捉和重新抛出没有任何的意义，进一步说，就跟没有使用异常处理的性质一样。

为了避免以上的问题，我们应该采用如下的方式

```cs
try
{
  // do stuff that might throw an exception
}
catch(Exception e)
{
  throw;
}
```

或者

```cs
try
{
  // do stuff that might throw an exception
}
catch(Exception ex)
{
  throw new Exception("my custom exception",ex);
}
```

即要么抛出异常，要么处理异常。

所以得出结论

* 1：如果你是第一种方法抛出异常，无需记录 `error log`.

* 2：如果你是第二种方法抛出异常，在处理异常的同时记录 `error log`

* 3：如果你不是上面两种方式抛出异常，修改你的代码，符合抛出异常的规范

Qustion: 写日志时，是把 `ex`对象传入日志里，还是`ex.Message`或`ex.StackTrace`?

先看下我们 `log4net`的配置文件：

```xml
<!--输出格式-->
<conversionPattern value="Description:%message%n Exception:%exception %n"/>
```
`message` 是 `log4net` 内置的变量，含义如下：
> Used to output the application supplied message associated with the logging event.

`exception` 是 `log4net` 内置的变量，含义如下：

> Used to output the exception passed in with the log message.

>If an exception object is stored in the logging event it will be rendered into the pattern output with a trailing newline. 

> If there is no exception then nothing will be output and no trailing newline will be appended. 

> It is typical to put a newline before the exception and to have the exception as the last data in the pattern.

我们调用实现的 `Error`接口代码时
```cs
 Logger.Instance.Error("Fail to do something...",ex);
```
得出结论：

1. 第一个参数传给`log4net`系统变量`message` 打印出来，最好打印一些有意义的语句，像上面的例子一样，打印

出具体什么事情失败了。

2. 第二个参数传给`log4net` 的 `LoggingEvent`对象，然后通过 `ExceptionObject`将整个`exception`渲染出来，并会自动换行，

所以我们直接传 `ex`对象即可。

格式：我们可以用， Fail to + <动词> + <事件> ，简单明了。

```
Fail to write file to disk, not enough space.
```

### 针对 Info Log

如果上面说的`try...catch(){}`使用合理的话，大部分的程序错误都能够被捕获到，并且结合错误日志可以分析错误的来源。

有时侯，我们还想关注程序执行的关键信息，这个时候，`Info log`就上场了。

`Info` ，顾名思义，信息。不是为了打印错误而生，而是为了记录信息，`Info`打印没有固定的标准，正因为如此，才更考验

程序员的功力，少一行看不出，多一行累赘，所以在写 `log` 之前最好先想好自己需要哪些信息。

尽管 `Info` 日志的打印因人而异，我们还是可以定一些基本要求：

1. 所有的输入输出，包括收消息和发消息都要求输出日志

2. 关键控制点必须输出日志

3. 调用底层或第三方软件，必须输出日志，而且对不可靠底层，必须加上 `begin/end` 两行日志，正好可以了解第三方执行情况

4. 第三方或数据库系统处理时间必须输出日志，以利于维护时快速定位性能问题

5. 不要在循环中打印日志

格式：

<动词> + <事件>

* 动词开头大写：更易被注意到，动词写前面而不是后面,可以让阅读者更快地感知主题。

* 注意时态：区分事情前后，读起来也更自然。发生在事情之前的动词用进行时(常为ing后缀)，发生在事情之后的用过去式(常为ed后缀)。

* 尽量选用不同的动词：搜索日志会比较方便，否则会匹配到大量字面相同但实则不同的日志。

```
Start to create a remote object...
Create a remote object,{remote object info here} 
Waiting remote to response back...{waiting time here}
Destroyed remote object...{remote object success released}
```

### 最后

上面的日志是针对具体的信息内容，除此之外，我们还会打印日志的时间，地点，人物。结合起来定位问题可以答到事倍功半的效果。

```xml
<conversionPattern value="**Datetime:【%date】 %n Level:【%-5level%】 %n HostName:%property{log4net:HostName} %n UserName:%username />
```
