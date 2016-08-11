#任务与gen_tcp

  1. 回显服务器
  2. 任务
  3. 任务主管

本章,我们将学习如何使用Erlang的`:gen_tcp`模块来处理请求.这是一个探索Elixir的`Task`模块的好机会.在后面的章节我们将扩展我们的服务器,让它能够执行命令.

#回显服务器(Echo server)

我们将实现一个回显服务器,作为我们的TCP服务器的开始.它会简单地发送一个回复,内容就是在请求中收到的文本.我们将缓慢地升级服务器,直到它被监督并准备好处理多重连接.

一个TCP服务器,概括地说,会有如下表现:

  1. 监听一个端口,直到端口可用,并获得套接字
  2. 在那个端口等待客户端连接并接受它
  3. 读取客户端请求并进行回复

让我们来实现这些步骤.进入`apps/kv_server`应用,打开`lib/kv_server.ex`,并添加下列函数:

```
require Logger

def accept(port) do
  # 下列选项的意思是:
  #
  # 1. `:binary` - 以二进制数接受数据 (而非列表)
  # 2. `packet: :line` - 逐行接收数据
  # 3. `active: false` - 阻塞在 `:gen_tcp.recv/2` 直到数据可用
  # 4. `reuseaddr: true` - 允许我们重用数据当监听器崩溃时
  {:ok, socket} = :gen_tcp.listen(port,
                    [:binary, packet: :line, active: false, reuseaddr: true])
  Logger.info "Accepting connections on port #{port}"
  loop_acceptor(socket)
end

defp loop_acceptor(socket) do
  {:ok, client} = :gen_tcp.accept(socket)
  serve(client)
  loop_acceptor(socket)
end

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

我们将通过调用`KVServer.accept(4040)`来开启我们的服务器,这里的4040是端口.`accept/1`的第一步就是监听端口,直到套接字可用,然后调用`loop_acceptor/1`.`loop_acceptor/1`只是一个接收客户端连接的循环.对于每个被接收的连接,我们调用`serve/1`.

`serve/1`是另一个循环,它从套接字中读取一行并将那些行写回套接字中.注意`serve/1`函数使用了管道操作符`|>`来表示这个操作流.管道操作符的左边会被执行,并将结果作为右边函数的第一个参数.上面的例子:

```
socket |> read_line() |> write_line(socket)
```

等同于:

```
write_line(read_line(socket), socket)
```

`read_line/1`通过`:gen_tcp.recv/2`实现了从套接字中接收数据,而`write_line/2`使用`:gen_tcp.send/2`向套接字中写入.

这些就是我们实现回显服务器所需的一切.让我们来试一试!

通过`iex -S mix`在`kv_server`应用中运行一个IEx会话.在IEx中,运行:

```
iex> KVServer.accept(4040)
```

现在服务器已经运行了,你会发现控制台阻塞了.让我们使用一个`telnet`客户端来访问我们的服务器.这些客户端在大多数操作系统中都是可用的,它们的命令行也是类似的:

```
$ telnet 127.0.0.1 4040
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
hello
hello
is it me
is it me
you are looking for?
you are looking for?
```

输入"hello",敲击回车,将返回"hello".完美!

我的telnet客户端可以通过输入`ctrl + ]`,输入`quit`,再敲击`<Enter>`来退出,但你的客户端可能步骤会有不同.

一旦你退出了telnet客户端,你会在IEx会话中看到一个错误:

```
** (MatchError) no match of right hand side value: {:error, :closed}
    (kv_server) lib/kv_server.ex:41: KVServer.read_line/1
    (kv_server) lib/kv_server.ex:33: KVServer.serve/1
    (kv_server) lib/kv_server.ex:27: KVServer.loop_acceptor/1
```

这是因为我们期望从`:gen_tcp.recv/2`中获得数据,但客户端关闭了连接.我们将在之后对服务器的修改中更好地处理这种情形.

现在,这里有一个更加重要的bug需要我们去修复:当我们的TCP接收器崩溃时会发生什么?因为这里没有监督,所以服务器死亡后,我们不能再处理更多请求,因为它不会重启.这就是为什么我们必须将服务器移动到一个监督树上.

#任务(Tasks)

我们已经学习了代理,通用服务器,以及主管.它们都需要处理多重的消息或消息状态.但是当我们只需要执行一些任务时应该使用什么呢?

Task模块提供了这个功能.例如,它有一个`start_link/3`函数用于接收模块,函数和参数,允许我们运行一个给定的函数作为监督树的一部分.

来试一下.打开`lib/kv_server.ex`,将`start/2`函数中的主管修改成:

```
def start(_type, _args) do
  import Supervisor.Spec

  children = [
    worker(Task, [KVServer, :accept, [4040]])
  ]

  opts = [strategy: :one_for_one, name: KVServer.Supervisor]
  Supervisor.start_link(children, opts)
end
```

通过这个改动,我们将`KVServer.accept(4040)`作为一个工人来运行.我们一直在编写端口,而等一下我们将会讨论它可以被如何改动.

现在服务器是监督树的一部分了,它会在我们运行应用时自动启动.在终端中输入`mix run --no-halt`,并再次使用`telnet`客户端来确认一切正常:

```
$ telnet 127.0.0.1 4040
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
say you
say you
say me
say me
```

正常!如果你杀死客户端,导致整个服务器崩溃,你会看到另一个立刻启动了.然而,它是规模化(scale)的吗?

试着同时连接两个telnet客户端.当你这么做时,你会发现,第二个客户端没有回声:

```
$ telnet 127.0.0.1 4040
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
hello
hello?
HELLOOOOOO?
```

它似乎完全不能工作.原因是我们接受连接和处理请求是在同一个进程中.当一个客户端连接上,我们就不能再接受其它客户端了.

#任务主管(Task Supervisor)

为了使我们的服务器能够处理同时连接,我们需要有一个接收器进程,它能够生成其它进程来处理请求.一个方法要修改一下:

```
defp loop_acceptor(socket) do
  {:ok, client} = :gen_tcp.accept(socket)
  serve(client)
  loop_acceptor(socket)
end
```

要使用`Task.start_link/1`,它类似于`Task.start_link/3`,但是它接收的是一个匿名函数而非模块,函数和参数:

```
defp loop_acceptor(socket) do
  {:ok, client} = :gen_tcp.accept(socket)
  Task.start_link(fn -> serve(client) end)
  loop_acceptor(socket)
end
```

我们直接在接收器进程中启动了一个链接任务.但我们已经犯过这种错误了.你还记得吗?

这类似于我们从注册表中直接调用`KV.Bucket.start_link/0`时所犯的错误.那意味着任何桶的失败都会导致整个注册表崩溃.

上述代码也有同样的缺陷:如果我们将`serve(client)`任务链接到接收器,一个处理请求时的崩溃将会传递给接收器,从而导致其它所有连接失败.

我们为注册表修正了这个问题,通过使用一个简单地一对一主管.我们在此将如法炮制,由于这个模式太常用于任务了,`Task`已经提供了一个方案:在我们的监督树上,可以使用一个简单的一对一主管,附带临时的工人.

让我们再次修改`start/2`,添加一个主管到我们的树上:

```
def start(_type, _args) do
  import Supervisor.Spec

  children = [
    supervisor(Task.Supervisor, [[name: KVServer.TaskSupervisor]]),
    worker(Task, [KVServer, :accept, [4040]])
  ]

  opts = [strategy: :one_for_one, name: KVServer.Supervisor]
  Supervisor.start_link(children, opts)
end
```

我们简单地启动了一个名为`KVSever.TaskSupervisor`的`Task.Supervisor`进程.记住,因为接受其任务依赖于该主管,所以主管必须先启动.

现在我们只需要修改`loop_acceptor/1`来使用`Task.Supervisor`处理每个请求:

```
defp loop_acceptor(socket) do
  {:ok, client} = :gen_tcp.accept(socket)
  {:ok, pid} = Task.Supervisor.start_child(KVServer.TaskSupervisor, fn -> serve(client) end)
  :ok = :gen_tcp.controlling_process(client, pid)
  loop_acceptor(socket)
end
```

你可能注意到我们添加了一行,`:ok = :gen_tcp.controlling_process(client, pid)`.这使得子进程成为了`client`套接字的"控制进程".如果不这样做的话,一旦接收器崩溃,它就会关闭所有客户端,因为套接字默认被绑定到了那个接收它们的进程上.

通过`mix run --no-halt`启动一个新的服务器,现在我们可以打开许多并发的telnet客户端了.你也会发现退出一个客户端不会导致接收器崩溃了.优秀!

这里是完整的回显服务器实现,在单独的模块中:

```
defmodule KVServer do
  use Application
  require Logger

  @doc false
  def start(_type, _args) do
    import Supervisor.Spec

    children = [
      supervisor(Task.Supervisor, [[name: KVServer.TaskSupervisor]]),
      worker(Task, [KVServer, :accept, [4040]])
    ]

    opts = [strategy: :one_for_one, name: KVServer.Supervisor]
    Supervisor.start_link(children, opts)
  end

  @doc """
  Starts accepting connections on the given `port`.
  """
  def accept(port) do
    {:ok, socket} = :gen_tcp.listen(port,
                      [:binary, packet: :line, active: false, reuseaddr: true])
    Logger.info "Accepting connections on port #{port}"
    loop_acceptor(socket)
  end

  defp loop_acceptor(socket) do
    {:ok, client} = :gen_tcp.accept(socket)
    {:ok, pid} = Task.Supervisor.start_child(KVServer.TaskSupervisor, fn -> serve(client) end)
    :ok = :gen_tcp.controlling_process(client, pid)
    loop_acceptor(socket)
  end

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
end
```

由于我们改变了主管的规则,我们不禁要问:我们的监督策略还正确吗?

在这种情况下,是正确的:如果接收器崩溃,不需要关闭现存的连接.换句话说,如果任务主管崩溃,也不需要关闭接收器.

下一章我们将开始解析客户端请求和发送回复,完成我们的服务器.
