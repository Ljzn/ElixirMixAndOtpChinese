#文档,测试与with

  1. 文档测试
  2. with
  3. 运行命令

本章,我们将实现能够解析我们在第一章中描述的命令的代码:

```
CREATE shopping
OK

PUT shopping milk 1
OK

PUT shopping eggs 3
OK

GET shopping milk
1
OK

DELETE shopping eggs
OK
```

解析完成后,我们将更新我们的服务器来调遣解析后的命令到`kv`应用中.

#文档测试(Doctests)

在语言主页,我们提到Elixir将文档当做语言中的一等公民.我们已经在本教程中多次探索了这个概念,通过`mix help`,或输入`h Enum`等其他模块在IEx控制台中.

本节,我们将使用文档测试来实现解析功能,它允许我们从文档中直接编写测试.这帮助我们给文档提供精确的代码样本.

让我们在`lib/kv_server/command.ex`中创建命令解析器,并以文档测试开头:

```
defmodule KVServer.Command do
  @doc ~S"""
  Parses the given `line` into a command.

  ## Examples

      iex> KVServer.Command.parse "CREATE shopping\r\n"
      {:ok, {:create, "shopping"}}

  """
  def parse(line) do
    :not_implemented
  end
end
```

文档测试是在文档字符串中定义的,通过四个空格的缩进之后跟着`iex>`语句来指定.如果一个命令跨越多行,你可以想在IEx中一样使用`...>`.预期的结果应该在`iex>`或`...>`的下一行开始,并以新的行或新的`iex>`前缀作为结尾.

还要注意的是我们使用`@doc ~S"""`来开始文档字符串.`~S`能够避免`\r\n`字符被转化成回车和换行,直到它们在测试中被执行.

要运行我们的文档测试,我们会在`test/kv_server/command_test.exs`中创建一个文件,并在测试中调用`doctest KVServer.Command`:

```
defmodule KVServer.CommandTest do
  use ExUnit.Case, async: true
  doctest KVServer.Command
end
```

运行这套测试,文档测试将会失败:

```
1) test doc at KVServer.Command.parse/1 (1) (KVServer.CommandTest)
   test/kv_server/command_test.exs:3
   Doctest failed
   code: KVServer.Command.parse "CREATE shopping\r\n" === {:ok, {:create, "shopping"}}
   lhs:  :not_implemented
   stacktrace:
     lib/kv_server/command.ex:11: KVServer.Command (module)
```

很好!

现在只需要让文档测试通过就行了.让我们来实现`parse/1`函数:

```
def parse(line) do
  case String.split(line) do
    ["CREATE", bucket] -> {:ok, {:create, bucket}}
  end
end
```

我们的实现是简单地用空格拆分命令行,然后匹配列表中的命令.使用`String.split/1`意味着我们的命令将会是空格不敏感的,开头和结尾的空格是无关紧要的,单词间连续的空格也是一样.让我们添加一些新的文档测试,来测试其它命令:

```
@doc ~S"""
Parses the given `line` into a command.

## Examples

    iex> KVServer.Command.parse "CREATE shopping\r\n"
    {:ok, {:create, "shopping"}}

    iex> KVServer.Command.parse "CREATE  shopping  \r\n"
    {:ok, {:create, "shopping"}}

    iex> KVServer.Command.parse "PUT shopping milk 1\r\n"
    {:ok, {:put, "shopping", "milk", "1"}}

    iex> KVServer.Command.parse "GET shopping milk\r\n"
    {:ok, {:get, "shopping", "milk"}}

    iex> KVServer.Command.parse "DELETE shopping eggs\r\n"
    {:ok, {:delete, "shopping", "eggs"}}

Unknown commands or commands with the wrong number of
arguments return an error:

    iex> KVServer.Command.parse "UNKNOWN shopping eggs\r\n"
    {:error, :unknown_command}

    iex> KVServer.Command.parse "GET shopping\r\n"
    {:error, :unknown_command}

"""
```

现在轮到你来让测试通过!你完成之后,可以对比一下我们的解决方案:

```
def parse(line) do
  case String.split(line) do
    ["CREATE", bucket] -> {:ok, {:create, bucket}}
    ["GET", bucket, key] -> {:ok, {:get, bucket, key}}
    ["PUT", bucket, key, value] -> {:ok, {:put, bucket, key, value}}
    ["DELETE", bucket, key] -> {:ok, {:delete, bucket, key}}
    _ -> {:error, :unknown_command}
  end
end
```

注意我们是如何优雅地解析命令的,不需要添加一大堆 的`if/else`从句来检查命令名和参数数量!

最后,你可能会发现每个文档测试都被认为是不同的测试,因为我们这套测试最后报告了7个测试.这是因为ExUnit是这样辨认两个不同测试的定义的:

```
iex> KVServer.Command.parse "UNKNOWN shopping eggs\r\n"
{:error, :unknown_command}

iex> KVServer.Command.parse "GET shopping\r\n"
{:error, :unknown_command}
```

中间没有隔一行的话,ExUnit就会将其编译为一个测试:

```
iex> KVServer.Command.parse "UNKNOWN shopping eggs\r\n"
{:error, :unknown_command}
iex> KVServer.Command.parse "GET shopping\r\n"
{:error, :unknown_command}
```

你可以阅读`ExUnit.DocTest`文档来获取更多关于文档测试的内容.

#with

现在我们能够解析命令了,我们终于可以开始实现运行命令的逻辑了.让我们为这个函数添加一个存根定义:

```
defmodule KVServer.Command do
  @doc """
  Runs the given command.
  """
  def run(command) do
    {:ok, "OK\r\n"}
  end
end
```

在我们实现这个函数之前,让我们修改服务器,使其开始使用我们新的`parse/1`和`run/1`函数.记住,我们的`read_line/1`函数会在客户端关闭套接字时崩溃,所以让我们也抓住机会修复它.打开`lib/kv_server.ex`:

```
defp serve(socket) do
  socket
  |> read_line()
  |> write_line(socket)

  serve(socket)
end

defp read_line(socket) do
  {:ok, data} = :gen_tcp.recv(socket, 0)
  data
end

defp write_line(line, socket) do
  :gen_tcp.send(socket, line)
end
```

替换成:

```
defp serve(socket) do
  msg =
    case read_line(socket) do
      {:ok, data} ->
        case KVServer.Command.parse(data) do
          {:ok, command} ->
            KVServer.Command.run(command)
          {:error, _} = err ->
            err
        end
      {:error, _} = err ->
        err
    end

  write_line(socket, msg)
  serve(socket)
end

defp read_line(socket) do
  :gen_tcp.recv(socket, 0)
end

defp write_line(socket, {:ok, text}) do
  :gen_tcp.send(socket, text)
end

defp write_line(socket, {:error, :unknown_command}) do
  # Known error. Write to the client.
  :gen_tcp.send(socket, "UNKNOWN COMMAND\r\n")
end

defp write_line(_socket, {:error, :closed}) do
  # The connection was closed, exit politely.
  exit(:shutdown)
end

defp write_line(socket, {:error, error}) do
  # Unknown error. Write to the client and exit.
  :gen_tcp.send(socket, "ERROR\r\n")
  exit(error)
end
```

启动我们的服务器,现在我们可以向它发送命令.现在我们可以得到两个不同的回复:当命令已知时回复"OK",否则回复"UNKNOWN COMMAND":

```
$ telnet 127.0.0.1 4040
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
CREATE shopping
OK
HELLO
UNKNOWN COMMAND
```

这意味着我们的实现已经朝着正确的方向运行,但是这看起来不太优雅,对吗?

之前的实现使用了资源管线,使得逻辑很清晰.然而,现在我们需要处理不同的错误代码,我们的服务器逻辑嵌套在了许多`case`调用中.

幸运的是,Elixir v1.2引入了一个叫做`with`的结构,它能够简化像上面那样的代码.让我们用它来重写`server/1`函数吧:

```
defp serve(socket) do
  msg =
    with {:ok, data} <- read_line(socket),
         {:ok, command} <- KVServer.Command.parse(data),
         do: KVServer.Command.run(command)

  write_line(socket, msg)
  serve(socket)
end
```

好多了!明智的语法,`with`的理解和`for`很类似.`with`将会获取`<-`右边的返回值,并与左边进行模式匹配.如果匹配成功,`with`会进入下一个表达式.如果匹配失败,未匹配的值将会被返回.

换句话说,我们将`case/2`中的每个表达式转化成了`with`中的步骤.只要任何一步中返回值不能匹配`{:ok, x}`,`with`就会跳出,并返回未匹配的值.

你可在我们的文档中获取更多关于`with`的信息.

#运行命令

最后一步是实现`KVServer.Command.run/1`,来使`:kv`应用运行解析后的命令.它的实现如下所示:

```
@doc """
Runs the given command.
"""
def run(command)

def run({:create, bucket}) do
  KV.Registry.create(KV.Registry, bucket)
  {:ok, "OK\r\n"}
end

def run({:get, bucket, key}) do
  lookup bucket, fn pid ->
    value = KV.Bucket.get(pid, key)
    {:ok, "#{value}\r\nOK\r\n"}
  end
end

def run({:put, bucket, key, value}) do
  lookup bucket, fn pid ->
    KV.Bucket.put(pid, key, value)
    {:ok, "OK\r\n"}
  end
end

def run({:delete, bucket, key}) do
  lookup bucket, fn pid ->
    KV.Bucket.delete(pid, key)
    {:ok, "OK\r\n"}
  end
end

defp lookup(bucket, callback) do
  case KV.Registry.lookup(KV.Registry, bucket) do
    {:ok, pid} -> callback.(pid)
    :error -> {:error, :not_found}
  end
end
```

这个实现很简单:我们只需要派遣到`KV.Registry`服务器,它是我们在`:kv`应用启动时注册的.因为我们的`:kv_server`依赖于`:kv`应用,所以完全可以依赖它所提供的服务器/服务.

注意到我们也定义了一个名为`lookup/2`的私有函数来完成一个常用功能:搜索桶,如果存在就返回它的`pid`,否则返回`{:error, :not_found}`.

此外,由于我们现在返回的是`{:error, :not_found}`,我们应该修改`KV.Server`中的`write_line/2`函数使之也能来打印这个错误:

```
defp write_line(socket, {:error, :not_found}) do
  :gen_tcp.send(socket, "NOT FOUND\r\n")
end
```

我们的服务器功能基本完成了!我们只需要添加测试.这一次,我们把测试留到最后,因为有一些重要的决定要做.

`KVServer.Command.run/1`的实现是直接发送命令到由`:kv`应用注册的`KV.Registry`服务器.这意味着这个服务器是全局的,如果我们有两个测试同时发送信息给它,我们的测试将会相互冲突(很可能失败).我们需要决定是使用相互独立且能同步运行的单元测试,还是运行在全局状态顶部的集成测试,但是每次测试就要调用应用的全栈.

目前我们只写过单元测试,而且是直接测试单个模块.然而,为了使`KVServer.Command.run/1`能像一个单元一样被测试,我们需要改变它的实现,不再直接发送命令到`KV.Registry`进程,而是传送一个作为参数的服务器.这意味着我们需要改变`run`的签名到`def run(command, pid)`,以及对`:create`命令的实现:

```
def run({:create, bucket}, pid) do
  KV.Registry.create(pid, bucket)
  {:ok, "OK\r\n"}
end
```

当对`KVServer.Command`进行测试时,我们需要启动一个`KV.Registry`的实例,类似于我们在`apps/kv/test/kv/registry_test.exs`中做的那样,并将其作为一个参数传送给`run/2`.

这已经成为我们一直在测试中使用的方法,它的优点是:

  \1. 我们的实现不会与任何特定的服务器名耦合
  \2. 我们可以保持同步运行测试,因为这里没有共用状态

然而,它的缺点是我们的API为了容纳所有的外部参数而变得非常大.

替代方案是编写集成测试,它依赖于全局服务器名来使用整个堆栈,从TCP服务器到桶.集成测试的缺点是它们会比单元测试慢得多,因此它们必须节制地使用.例如,我们不应该使用集成测试在我们的命令解析实现中来测试一个边界情况.

现在我们将编写一个集成测试.集成测试会使用一个TCP客户端来发送命令到我们的服务器,并断言我们将得到预期的回复.

让我们在`test/kv_server_test.exs`中实现如下所示的集成测试:

```
defmodule KVServerTest do
  use ExUnit.Case

  setup do
    Application.stop(:kv)
    :ok = Application.start(:kv)
  end

  setup do
    opts = [:binary, packet: :line, active: false]
    {:ok, socket} = :gen_tcp.connect('localhost', 4040, opts)
    {:ok, socket: socket}
  end

  test "server interaction", %{socket: socket} do
    assert send_and_recv(socket, "UNKNOWN shopping\r\n") ==
           "UNKNOWN COMMAND\r\n"

    assert send_and_recv(socket, "GET shopping eggs\r\n") ==
           "NOT FOUND\r\n"

    assert send_and_recv(socket, "CREATE shopping\r\n") ==
           "OK\r\n"

    assert send_and_recv(socket, "PUT shopping eggs 3\r\n") ==
           "OK\r\n"

    # GET returns two lines
    assert send_and_recv(socket, "GET shopping eggs\r\n") == "3\r\n"
    assert send_and_recv(socket, "") == "OK\r\n"

    assert send_and_recv(socket, "DELETE shopping eggs\r\n") ==
           "OK\r\n"

    # GET returns two lines
    assert send_and_recv(socket, "GET shopping eggs\r\n") == "\r\n"
    assert send_and_recv(socket, "") == "OK\r\n"
  end

  defp send_and_recv(socket, command) do
    :ok = :gen_tcp.send(socket, command)
    {:ok, data} = :gen_tcp.recv(socket, 0, 1000)
    data
  end
end
```

我们的集成测试检查了所有的服务器接口,包括未知命令和未找到错误.因为是在处理ETS表格和链接进程,所以不必关闭套接字.一旦测试进程退出,套接字会自动关闭.

这一次,因为我们的测试依赖于全局数据,所以我们没有将`async: true`传送给`use ExUnit.Case`.而且,为了保证我们的测试始终在一个干净的状态,在每个测试之前我们停止再启动了`:kv`应用.事实上,停止`:kv`应用会在终端打印一个警告:

```
18:12:10.698 [info] Application kv exited: :stopped
```

为了避免在测试过程中打印日志,ExUnit提供了一个叫做`:capture_log`的干净特性.通过在每次测试前设置`@tag :capture_log`,或者为整个测试设置`@moduletag :capture_log`,在测试运行时,ExUnit会自动捕获日志中的任何东西.如果测试失败,捕获的日志会被打印在ExUnit报告旁边.

启动之前,添加如下调用:

```
@moduletag :capture_log
```

当测试崩溃时,你会看到如下报告:

```
1) test server interaction (KVServerTest)
   test/kv_server_test.exs:17
   ** (RuntimeError) oops
   stacktrace:
     test/kv_server_test.exs:29

   The following output was logged:

   13:44:10.035 [info]  Application kv exited: :stopped
```

从这个简单的集成测试中,我们可以知道为什么集成测试可能很慢.不止因为这种测试不能同步运行,还因为要求停止再启动`:kv`应用这种昂贵的启动配置.

最后,应当由你和你的团队来找到适用于你的应用的最好的测试策略.你需要平衡代码质量,信心,和测试套件的运行时.例如,最开始我们可能只用集成测试来测试服务器,但是如果服务器在之后的发布中持续成长,或者它成为了一个频繁发生bug的应用的一部分,那么考虑将其打碎并编写更多加强的比集成测试轻量得多的单元测试就变得非常重要.

在下一章,我们终于要通过添加一个桶路由机制来使得我们的系统成为分布式的.我们也将学习应用配置.
