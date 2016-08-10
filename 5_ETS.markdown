#ETS

  1. ETS做缓存
  2. 竞态条件

每当我们需要查找一个桶,我们就要发送一个信息给注册表.这时我们的注册表会被多个进程并发访问,就会遇到瓶颈.

本章我们会学习ETS(Erlang条件存储)以及如何使用它作为缓存机制.

> 警告!不要贸然地使用ETS做缓存!查看日志并分析你的应用表现,确定那一部分是瓶颈,这样你就可以知道是否应该使用缓存,以及缓存什么.本章仅仅是一个如何使用ETS做缓存的例子.

#ETS做缓存

ETS允许我们存储任何Elixir条件到一个内存中的表格.操作ETS表格需要通过Erlang的`:ets`模块:

```
iex> table = :ets.new(:buckets_registry, [:set, :protected])
8207
iex> :ets.insert(table, {"foo", self})
true
iex> :ets.lookup(table, "foo")
[{"foo", #PID<0.41.0>}]
```

创建一个ETS表格时,需要两个参数:表格名以及一些选项.通过这些选项,我们传送了表格类型和它的访问规则.我们已经选择了`:set`类型,它意味着键不可以被复制.我们也将表格访问设置成了`:protected`,意味着只有创建了这个表格的进程可以写入它,但所有进程都可以从表中读取.这些都是默认值,所以我们将在之后跳过它们.

ETS表格可以被命名,允许我们通过名称来访问:

```
iex> :ets.new(:buckets_registry, [:named_table])
:buckets_registry
iex> :ets.insert(:buckets_registry, {"foo", self})
true
iex> :ets.lookup(:buckets_registry, "foo")
[{"foo", #PID<0.41.0>}]
```

让我们修改`KV.Registry`来使用ETS表格.由于我们的注册表需要一个名字作为参数,我们可以用相同的名字命名ETS表格.ETS名与进程名存储在不同的位置,所以它们不会冲突.

打开`lib/kv/registry.ex`,让我们来改变它的实现.我们已经为源代码的修改添加了注释:

```
defmodule KV.Registry do
  use GenServer

  ## Client API

  @doc """
  Starts the registry with the given `name`.
  """
  def start_link(name) do
    # 1. 传送名字给GenServer的init
    GenServer.start_link(__MODULE__, name, name: name)
  end

  @doc """
  Looks up the bucket pid for `name` stored in `server`.

  Returns `{:ok, pid}` if the bucket exists, `:error` otherwise.
  """
  def lookup(server, name) when is_atom(server) do
    # 2. 查找将直接在ETS中运行,不需要访问服务器
    case :ets.lookup(server, name) do
      [{^name, pid}] -> {:ok, pid}
      [] -> :error
    end
  end

  @doc """
  Ensures there is a bucket associated to the given `name` in `server`.
  """
  def create(server, name) do
    GenServer.cast(server, {:create, name})
  end

  @doc """
  Stops the registry.
  """
  def stop(server) do
    GenServer.stop(server)
  end

  ## Server callbacks

  def init(table) do
    # 3. 我们已经用ETS表格取代了名称映射
    names = :ets.new(table, [:named_table, read_concurrency: true])
    refs  = %{}
    {:ok, {names, refs}}
  end

  # 4. 之前用于查找的handle_call回调已经删除

  def handle_cast({:create, name}, {names, refs}) do
    # 5. 对ETS表格进行读写,而非对映射
    case lookup(names, name) do
      {:ok, _pid} ->
        {:noreply, {names, refs}}
      :error ->
        {:ok, pid} = KV.Bucket.Supervisor.start_bucket
        ref = Process.monitor(pid)
        refs = Map.put(refs, ref, name)
        :ets.insert(names, {name, pid})
        {:noreply, {names, refs}}
    end
  end

  def handle_info({:DOWN, ref, :process, _pid, _reason}, {names, refs}) do
    # 6. 从ETS表中删除而不是从映射中
    {name, refs} = Map.pop(refs, ref)
    :ets.delete(names, name)
    {:noreply, {names, refs}}
  end
  
  def handle_info(_msg, state) do
    {:noreply, state}
  end
end
```

注意到在修改之前,`KV.Reigstry.lookup/2`会将请求发送给服务器,但现在会直接从ETS表格中读取,改表格会被所有进程分享.这就是我们实现的缓存机制背后的主要原理.

为了使缓存机制运行,ETS表格需要有让对其的访问`:protected`(默认),这样所有客户端都可以从中读取,而只有`KV.Registry`进程能写入.我们也在创建表格时设置了`read_concurrency: true`,为普通的并发读取操作的脚本做了表格优化.

上诉修改破坏了我们的测试,因为之前为所有操作使用的是注册表进程的pid,而现在注册表查找要求的是ETS表格名.然而,由于注册表进程和ETS表格有着相同的名字,就很好解决这个问题.将`test/kv/registry_test.exs`中的设置函数修改为:

```
setup context do
  {:ok, _} = KV.Registry.start_link(context.test)
  {:ok, registry: context.test}
end
```

我们修改了`setup`,仍然有一些测试失败了.你可能会注意到每次测试的结果不一致.例如,"生成桶"的测试:

```
test "spawns buckets", %{registry: registry} do
  assert KV.Registry.lookup(registry, "shopping") == :error

  KV.Registry.create(registry, "shopping")
  assert {:ok, bucket} = KV.Registry.lookup(registry, "shopping")

  KV.Bucket.put(bucket, "milk", 1)
  assert KV.Bucket.get(bucket, "milk") == 1
end
```

可能会在这里失败:

```
{:ok, bucket} = KV.Registry.lookup(registry, "shopping")
```

为什么这条会失败,我们明明已经在上一条代码中创建了桶?

原因就是,出于教学目的,我们犯了两个错误:

  \1. 我们贸然做了优化(通过添加这个缓存层)
  \2. 我们使用了`cast/2`(应该使用`call/2`)

#竞态条件?

使用Elixir开发并不能使你的代码魔兽竞态条件影响.然而,Elixir中默认的不共用任何事物的简单概念使得我们更容易发现产生竞态条件的原因.

在发出操作和从ETS表格观察到改变之间会有延迟,导致了我们测试的失败.下面是我们希望看到的:

  \1. 我们调用`KV.Rgistry.create(registry, "shopping")`
  \2. 注册表创建了桶并更新了缓存表格
  \3. 我们使用`KV.Registry.lookup(registry, "shopping")`从表格中获取信息
  \4. 返回`{:ok, bucket}`

然而,由于`KV.Registry.create/2`是一个投掷操作,这条命令将会在我们真正写入表格之前返回!换句话或,实际发生的是:

  \1. 我们调用`KV.Rgistry.create(registry, "shopping")`
  \2. 我们使用`KV.Registry.lookup(ets, "shopping")`从表格中获取信息
  \3. 返回`:error`
  \4. 注册表创建了桶并更新了缓存表格

为了修正这个错误,我们需要使得`KV.Registry.create/2`变为异步的,通过使用`call/2`代替`cast/2`.这就保证了客户端只会在对表格的改动发生过后继续.让我们修改函数,以及它的回调:

```
def create(server, name) do
  GenServer.call(server, {:create, name})
end

def handle_call({:create, name}, _from, {names, refs}) do
  case lookup(names, name) do
    {:ok, pid} ->
      {:reply, pid, {names, refs}}
    :error ->
      {:ok, pid} = KV.Bucket.Supervisor.start_bucket
      ref = Process.monitor(pid)
      refs = Map.put(refs, ref, name)
      :ets.insert(names, {name, pid})
      {:reply, pid, {names, refs}}
  end
end
```

我们简单地将回调从`handle_cast/2`改为了`handle_call/3`,并回复被创建了的桶的pid.通常来说,Elixir开发者更喜欢使用`call/2`而不是`cast/2`,因为它也提供了背压(你会被挡住直到得到回复).在不必要时使用`cast/2`也可以被看做是贸然的优化.

让我们再次运行测试.这一次,我们会传送`--trace`选项:

```
$ mix test --trace
```

`--trace`选项在你的测试死锁或遇到竞态条件时很有用,因为它会异步运行所有测试(`async: true`无效)并展示每个测试的详细信息.这一次我们应该会得到一两个断断续续的失败:

```
1) test removes buckets on exit (KV.RegistryTest)
   test/kv/registry_test.exs:19
   Assertion with == failed
   code: KV.Registry.lookup(registry, "shopping") == :error
   lhs:  {:ok, #PID<0.109.0>}
   rhs:  :error
   stacktrace:
     test/kv/registry_test.exs:23
```

根据错误信息,我们期望桶不再存在,但它仍在那儿!这个问题与我们刚才解决的正相反:之前在命令创建桶与更新表格之间存在延迟,现在是在桶进程死亡与它在表中的记录被删除之间存在延迟.

不走运的是这一次我们不能简单地将`handle_info/2`这个负责清洁ETS表格的操作,改变为一个异步操作.相反我们需要找到一个方法来保证注册表已经处理了`:DOWN`通知的发送,当桶崩溃时.

简单的方法是发送一个异步请求给注册表:因为信息会被按顺序处理,如果注册表回复了一个在`Agent.stop`调用之后的发送的请求,就意味着`:DOWN`消息已经被处理了.让我们创建一个"bogus"桶,它是一个异步请求,在每个测试中排在`Agent.stop`之后:

```
test "removes buckets on exit", %{registry: registry} do
  KV.Registry.create(registry, "shopping")
  {:ok, bucket} = KV.Registry.lookup(registry, "shopping")
  Agent.stop(bucket)

  # Do a call to ensure the registry processed the DOWN message
  _ = KV.Registry.create(registry, "bogus")
  assert KV.Registry.lookup(registry, "shopping") == :error
end

test "removes bucket on crash", %{registry: registry} do
  KV.Registry.create(registry, "shopping")
  {:ok, bucket} = KV.Registry.lookup(registry, "shopping")

  # Kill the bucket and wait for the notification
  Process.exit(bucket, :shutdown)

  # Wait until the bucket is dead
  ref = Process.monitor(bucket)
  assert_receive {:DOWN, ^ref, _, _, _}

  # Do a call to ensure the registry processed the DOWN message
  _ = KV.Registry.create(registry, "bogus")
  assert KV.Registry.lookup(registry, "shopping") == :error
end
```

我们的测试现在可以(一直)通过了!

该总结一下我们的优化章节了.我们使用了ETS作为缓存机制,一人写万人读.更重要的是,一但数据可以被同步读取,就要当心竞态条件.

下一章我们将讨论外部和内部的依赖,以及Mix如何帮助我们管理巨型代码库.
