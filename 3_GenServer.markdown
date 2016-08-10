#通用服务器

  1. 我们的第一个通用服务器
  2. 测试一个通用服务器
  3. 监控的需要
  4. `call`,`cast`还是`info`?
  5. 监控器还是链接?

之前的章节里我们用代理来代表桶.在第一章,我们曾指出想要给每个桶命名,之后我们可以这样做:

```
CREATE shopping
OK

PUT shopping milk 1
OK

GET shopping milk
1
OK
```

因为代理也是进程,所以每个桶有一个进程辨识符(pid)却没有名字.我们已经在进程章节中学习过名称注册,你可以在这里使用它.例如,我们可以这样创造一个桶:

```
iex> Agent.start_link(fn -> %{} end, name: :shopping)
{:ok, #PID<0.43.0>}
iex> KV.Bucket.put(:shopping, "milk", 1)
:ok
iex> KV.Bucket.get(:shopping, "milk")
1
```

然而,这是一个糟糕的主意!在Elixir中进程名必须是原子,这意味着我们需要将桶名(通常接收自外部客户端)转换成原子,而**我们永远不应该将用户输入转换成原子**.这是因为原子不支持垃圾回收.原子一旦创造出来,就永远不会被回收.从用户输入中生成原子意味着用户可以注入足够多的不同名字来耗尽我们的系统内存!

在实际中你会在耗尽内存之前达到Erlang虚拟机的原子上限,然后系统会被强制关闭.

没有滥用名称注册功能,我们创造了自己的_注册过程_.就是将桶名与桶进程的联系放到了一个映射中.

注册表需要保证词典总是最新的.例如,如果一个桶进程因为一个bug而崩溃了,注册表必须将词典清理干净以避免提供陈旧的条目.在Elixir中,我们将此描述为注册表需要_监控_每个桶.

我们将使用通用服务器来创建一个能监控桶进程的注册表进程.GenServers是Elixir和OTP中用于构建通用服务器的go-to抽象.

#我们的第一个通用服务器

一个通用服务器由两部分实现:客户端API和服务器回调,要么是在一个模块中,要么是在两个不同的模块中实现的它们.客户端与服务器运行在不同的进程中,当函数被调用时,客户端将会传递信息给服务器.在此我们使用一个模块来定义服务器回调和客户端API.新建`lib/kv/registry.ex`文件:

```
defmodule KV.Registry do
  use GenServer

  ## Client API

  @doc """
  Starts the registry.
  """
  def start_link do
    GenServer.start_link(__MODULE__, :ok, [])
  end

  @doc """
  Looks up the bucket pid for `name` stored in `server`.

  Returns `{:ok, pid}` if the bucket exists, `:error` otherwise.
  """
  def lookup(server, name) do
    GenServer.call(server, {:lookup, name})
  end

  @doc """
  Ensures there is a bucket associated to the given `name` in `server`.
  """
  def create(server, name) do
    GenServer.cast(server, {:create, name})
  end

  ## Server Callbacks

  def init(:ok) do
    {:ok, %{}}
  end

  def handle_call({:lookup, name}, _from, names) do
    {:reply, Map.fetch(names, name), names}
  end

  def handle_cast({:create, name}, names) do
    if Map.has_key?(names, name) do
      {:noreply, names}
    else
      {:ok, bucket} = KV.Bucket.start_link
      {:noreply, Map.put(names, name, bucket)}
    end
  end
end
```

第一个函数是`start_link/3`,它开启了一个新的GenServer并传送了三个参数:

  1. 服务器回调将会在哪一个模块中实现,`__MODULE__`意为当前模块

  2. 初始参数,此时是原子`:ok`

  3. 选项列表,它有保存服务器名称等功能.现在我们传送一个空列表

你可以向GenServer发送两种请求:呼叫与投掷.呼叫是同步的,服务器**必须**回应.投掷是异步的,服务器不会回应.

接下来的两个函数,`lookup/2`和`create/2`负责发送这些请求到服务器.这些请求会出现在`handle_call/3`或`handle_cast/2`的第一个参数.本例中,我们分别使用`{:lookup, name}`和`{:create, name}`.请求通常以元组形式表达,这是为了在第一个参数的位置上提供不止一个"参数".通常元组的第一个元素是请求的动作,余下的元素是该动作的参数.

在服务器端,我们可以实现多种回调来保证服务器的初始化,终止和请求处理.这些回调是可选的,现在我们只实现了我们在乎的那些.

第一个回调是`init/1`,它得到了提供给`GenServer.start_link/3`的参数,并返回`{:ok, state}`,这里的状态是一个新映射.我们已经可以注意到`GenServer`API是如何使客户端/服务器隔离得更明显的.`start_link/3`发生在客户端,而`init/1`是运行在服务器上的回调.

我们必须实现一个`handle_call/3`回调来接收`call/2`的`request`,我们是从哪个进程收到请求的(`_from`),和当前服务器状态(`names`).`handle_call/3`回调返回了一个格式为`{:reply, reply, new_state}`的元组,`reply`是将要发送到客户端的内容,而`new_state`是新的服务器状态.

我们必须实现一个`handle_cast/2`回调来接收`cast/2`的`request`,和当前服务器状态(`names`).`handle_cast/2`回调返回了一个格式为`{:noreply, new_state}`的元组.

这里还有一些`handle_call/3`和`handle_cast/2`回调都可能返回的元组格式.我们还可以实现其它的回调,例如`terminate/2`和`code_change/3`.欢迎到full GenServer文档中学习更多相关内容.

现在,让我们来写一些测试来保证我们的GenServer运作正常.

#测试通用服务器

测试GenServer与测试代理差不多.我们将生成一个设置回调上的服务器,并在测试中使用它.创建一个名为`test/kv/registry_test.exs`的文件:

```
defmodule KV.RegistryTest do
  use ExUnit.Case, async: true

  setup do
    {:ok, registry} = KV.Registry.start_link
    {:ok, registry: registry}
  end

  test "spawns buckets", %{registry: registry} do
    assert KV.Registry.lookup(registry, "shopping") == :error

    KV.Registry.create(registry, "shopping")
    assert {:ok, bucket} = KV.Registry.lookup(registry, "shopping")

    KV.Bucket.put(bucket, "milk", 1)
    assert KV.Bucket.get(bucket, "milk") == 1
  end
end
```

我们的测试已经完成!

我们不需要明确地关闭注册,因为它将在测试结束时接收到一个`:shutdown`信号.尽管对于测试来说这种方法很正常,但如果需要将停止GenServer作为应用逻辑中的一部分,我们可以使用`GenServer.stop/1`函数:

```
## Client API

@doc """
Stops the registry.
"""
def stop(server) do
  GenServer.stop(server)
end
```

#监控的需要

我们的注册表快要完成了.唯一的问题就是注册表在桶停止或崩溃时有可能过时.让我们添加一个测试到`KV.Registry_Test`来引发这个bug:

```
test "removes buckets on exit", %{registry: registry} do
  KV.Registry.create(registry, "shopping")
  {:ok, bucket} = KV.Registry.lookup(registry, "shopping")
  Agent.stop(bucket)
  assert KV.Registry.lookup(registry, "shopping") == :error
end
```

这个测试会在最后一个断言失败,因为桶名仍然存在于注册表中,即使桶进程已经停止了.

为了修复这个bug,我们需要注册表监控每个由它生成的桶.设置好监视器之后,每当有桶退出,注册表就会收到一个通知,让我们清理词典.

让我们与监视器开始初次接触,通过运行`iex -S mix`来开启一个新的控制台:

```
iex> {:ok, pid} = KV.Bucket.start_link
{:ok, #PID<0.66.0>}
iex> Process.monitor(pid)
#Reference<0.0.0.551>
iex> Agent.stop(pid)
:ok
iex> flush
{:DOWN, #Reference<0.0.0.551>, :process, #PID<0.66.0>, :normal}
```

注意`Process.monitor(pid)`返回了一个独特的参照,我们能够用它来匹配即将到来的信息.在我们停止了代理之后,我们可以`flush/0`所有消息,发现收到了一个`:DOWN`信息,带有由监视器返回的确切的参照,提醒我们桶进程因为`:normal`原因退出了.

让我们重新实现服务器回调来修复bug并使测试通过.首先,我们会将GenServer状态修改成两个词典:一个包含`name -> pid`,另一个包含`ref -> name`.然后我们需要在`handle_cast/2`上监控桶,还需要实现`handle_info/2`回调来处理监控信息.完整的服务器回调实现如下所示:

```
## Server callbacks

def init(:ok) do
  names = %{}
  refs  = %{}
  {:ok, {names, refs}}
end

def handle_call({:lookup, name}, _from, {names, _} = state) do
  {:reply, Map.fetch(names, name), state}
end

def handle_cast({:create, name}, {names, refs}) do
  if Map.has_key?(names, name) do
    {:noreply, {names, refs}}
  else
    {:ok, pid} = KV.Bucket.start_link
    ref = Process.monitor(pid)
    refs = Map.put(refs, ref, name)
    names = Map.put(names, name, pid)
    {:noreply, {names, refs}}
  end
end

def handle_info({:DOWN, ref, :process, _pid, _reason}, {names, refs}) do
  {name, refs} = Map.pop(refs, ref)
  names = Map.delete(names, name)
  {:noreply, {names, refs}}
end

def handle_info(_msg, state) do
  {:noreply, state}
end
```

观察到,我们能大大改变服务器的实现,而不改变任何客户端接口.这是将服务器与客户端明确分离的好处之一.

最后,与其他回调不同,我们为`handle_info/2`定义了一个"全部捕获"从句,它抛弃了所有未知信息.想知道为什么,请看下一部分.

#`call`,`cast`还是`info`?

目前我们已经使用了三种回调:`handle_call/3`,`handle_cast/2`以及`handle_info/2`.它们适用于不同场景:

  1. `handle_call/3`必须用于同步任务.这应当是默认的选择,因为等待服务器回应是一个很有用的背压机制.

  2. `handle_cast/2`必须用于异步请求,当你不关心回复时.投掷甚至不保证服务器接收到信息,所以必须谨慎使用.例如,我们在本章中定义的`create/2`函数就应当使用`call/2`.出于教学目的我们使用了  `cast/2`.

  3. `handle_info/2`必须用于服务器可接收的所有不由`GenServer.call/2`或`GenServer.cast/2`发送的信息,包括由`send/2`发送的定期信息.监控`:DOWN`信息就是一个完美的例子.

因为任何信息,包括由`send/2`发送的,都传给了`handle_info/2`,服务器就有可能受到不想要的信息.因此,如果我们不定义捕获全部的从句,这些信息可能会使我们的注册表崩溃,因为没有从句可以匹配.

我们不需要为`handle_call/3`和`handle_cast/2`担心这个,因为这些请求只由`GenServer来完成`,所以未知信息很有可能是因为开发者的错误.

#监控器还是链接?

我们已经在进程那一章中学过链接了.现在,注册表完成了,你可能想知道:我们什么时候应该使用监控器,什么时候该使用链接?

链接是双向的.如果你将两个进程链接在一起,那么如果其中一个崩溃了,另一个也会崩溃(除非它捕获退出).监控器是单向的:只有监控进程会收到关于被监控者的消息.简而言之,当你想传递崩溃时就用链接,当你只想得到崩溃,退出等等的提示时就用监控器.

回到我们的`handle_cast/2`实现,你会看到注册表即链接又监控了桶:

```
{:ok, pid} = KV.Bucket.start_link
ref = Process.monitor(pid)
```

这是一个坏主意,因为我们不想桶的崩溃导致注册表崩溃!我们避免直接创建新的进程,而是将此责任交于监督者.如下一章我们将看到的,监督者依赖于链接,这解释了为什么在Elixir和OTP中基于链接的API(`spawn_link`,`start_link`等等)如此流行.
