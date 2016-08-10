#代理

  1. 状态问题
  2. 代理
  3. ExUnit回调
  4. 其它代理动作
  5. 代理中的客户端/服务器

本章我们将创建一个名为`KV.Bucket`的模块.这个模块的作用是以一种可以被其它进程读取和修改的方式存储我们的键值对.

如果你跳过了入门指导,或者你在很久以前读过,请再读一遍关于进程的那一章.我们将以其为起点.

#状态问题

Elixir是一门不可变语言,默认不分享任何东西.如果我们想要提供一个状态,在其中能够从多个地方创建桶的存取值,在Elixir中我们有两个主要选项:

  - 进程
  - ETS(Erlang长期存储)

我们已经讨论过了进程,而ETS是我们将在之后的教程中探索的新东西.我们很少直接操作进程,而是使用Elixir和OTP中的抽象概念:

  - 代理--状态的简单外壳
  - GenServer--"通用服务器"封装了状态(的进程),支持同步和异步调用,支持代码重载,等等.
  - GenEvent--"通用事件"管理将事件发布到多个处理程序.
  - 任务--允许生成进程并在之后悄悄获取结果的异步计算单元.

我们将在本教程中探索这些抽象概念.记住它们都是在进程之上使用VM提供的基础特性来实现的,例如`send`,`receive`,`spawn`和`link`.

#代理

代理是状态的简单包装.如果你只想用进程来保持状态,代理是最好的选择.让我们在项目内开始一个`iex`会话:

```
$ iex -S mix
```

使用代理:

```
iex> {:ok, agent} = Agent.start_link fn -> [] end
{:ok, #PID<0.57.0>}
iex> Agent.update(agent, fn list -> ["eggs" | list] end)
:ok
iex> Agent.get(agent, fn list -> list end)
["eggs"]
iex> Agent.stop(agent)
:ok
```

我们以一个空列表的初始状态最为代理的开始.我们更新了代理的状态,将我们的新元素添加到列表头.`Agent.update/3`的第二参数是一个函数,它的输入是代理的当前状态,而返回值是期望的新状态.最后,我们试图得到整个列表.`Agent.get/3`的第二个参数是一个函数,它将状态作为输入,而返回值是`Agent.get/3`本身将返回的值.完成了代理操作之后,我们可以调用`Agent.stop/3`来终止代理进程.

让我们使用代理来实现`KV.Bucket`.在开始实现之前,让我们先写一些测试.创建一个名为`test/kv/bucket_test.exs`的文件,内容是:

```
defmodule KV.BucketTest do
  use ExUnit.Case, async: true

  test "stores values by key" do
    {:ok, bucket} = KV.Bucket.start_link
    assert KV.Bucket.get(bucket, "milk") == nil

    KV.Bucket.put(bucket, "milk", 3)
    assert KV.Bucket.get(bucket, "milk") == 3
  end
end
```

我们的第一个测试开启了一个新的`KV.Bucket`,并对其进行了一些`get/2`和`put/3`操作,并断言了结果.我们不需要明确地停止代理,因为它与测试进程是相链接的,一旦测试结束代理会自动关闭.如果进程被命名,这就不管用了.

还要注意的是,我们传送了设置`async: true`给`ExUnit.Case`.这个设置使得该测试和其它开启了`:async`选项的测试可以并行运行.这对于使用我们机器上的多核心来提升测试速度非常有用.注意仅当测试不依赖或改变任何全局值时,才可以设置`:async`选项.例如,如果测试要求写入文件系统,注册进程,访问数据库,你就不能将其设为异步以避免出现测试间的竞争状态.

不论是否是异步的,我们的新测试显然会失败,因为没有函数被实现.

为了修正失败的测试,让我们创建一个内容如下的`lib/kv/bucket.ex`文件.在偷看下面的实现之前,大胆尝试自己使用代理来实现`KV.Bucket`模块.

```
defmodule KV.Bucket do
  @doc """
  Starts a new bucket.
  """
  def start_link do
    Agent.start_link(fn -> %{} end)
  end

  @doc """
  Gets a value from the `bucket` by `key`.
  """
  def get(bucket, key) do
    Agent.get(bucket, &Map.get(&1, key))
  end

  @doc """
  Puts the `value` for the given `key` in the `bucket`.
  """
  def put(bucket, key, value) do
    Agent.update(bucket, &Map.put(&1, key, value))
  end
end
```

我们使用映射来存储我们的键值.捕获操作符`&`在入门教程中介绍过了.

现在`KV.Bucket`模块已经定义好了,我们的测试应该会通过!你可以通过运行`mix test`来自己试一试.

#ExUnit回调

再继续前进,继续添加更多的特性到`KV.Bucket`中之前,让我们来讨论一下ExUnit回调.如你期望的那样,`KV.Bucket`的所有测试都要求在启动阶段开启一个桶,并在测试完成后停止它.幸运的是,ExUnit支持回调,使得我们能够跳过这些重复任务.

让我们重写测试案例,来使用回调:

```
defmodule KV.BucketTest do
  use ExUnit.Case, async: true

  setup do
    {:ok, bucket} = KV.Bucket.start_link
    {:ok, bucket: bucket}
  end

  test "stores values by key", %{bucket: bucket} do
    assert KV.Bucket.get(bucket, "milk") == nil

    KV.Bucket.put(bucket, "milk", 3)
    assert KV.Bucket.get(bucket, "milk") == 3
  end
end
```

我们已经先用`setup/1`宏来定义了一个设置回调.`setup/1`回调会在每个测试之前运行,在与该测试相同的进程中.

注意我们需要一个机制来讲`bucket`的pid由回调传递给测试.我们使用_测试内容_来做到它.当我们从回调中返回`{:ok, bucket: bucket}`,ExUnit会将元组的第二个元素(一个词典)融合进测试内容.测试内容是一个我们能够在测试定义中匹配的映射,它提供了获取块中的值得途径:

```
test "stores values by key", %{bucket: bucket} do
  # `bucket` 现在是设置块中的 bucket
end
```

你可以从`ExUnit.Case`模块和`ExUnit.Callback`文档中得到更多关于ExUnit案例和回调的内容.

#其它代理动作

除了获取值和更细代理状态,代理还允许我们通过调用`Agent.get_and_update/2`这个函数来同时实现取值与更新.让我们实现一个用于从桶中删除一个键并返回它当前值的函数`KV.Bucket.delete/2`:

```
@doc """
Deletes `key` from `bucket`.

Returns the current value of `key`, if `key` exists.
"""
def delete(bucket, key) do
  Agent.get_and_update(bucket, &Map.pop(&1, key))
end
```

现在轮到你来为上述功能编写测试!别忘了到`Agent`模块的文档中学习更多相关内容.

#代理中的客户端/服务器

在进入下一章之前,让我们来讨论一下代理中客户端与服务器的区别.让我们扩展一下刚才实现的`delete/2`函数:

```
def delete(bucket, key) do
  Agent.get_and_update(bucket, fn dict->
    Map.pop(dict, key)
  end)
end
```

我们传递给代理的函数中所发生的一切都是在代理进程中.这样,由于代理进程能接收并回应我们的信息,我们称代理进程为服务器.所有函数之外发生的都称为客户端.

这个区分很重要.如果有一个昂贵的动作要被执行,你必须考虑是在客户端还是服务器执行会更好.例如:

```
def delete(bucket, key) do
  Process.sleep(1000) # puts client to sleep
  Agent.get_and_update(bucket, fn dict ->
    Process.sleep(1000) # puts server to sleep
    Map.pop(dict, key)
  end)
end
```

当服务器执行一个漫长的操作时,所有对此服务器的其它请求都会被搁置,这有可能导致一些客户端超时.

在下一章我们将探索通用服务器,其中的客户端是相互隔离的,而服务器会被制作得更加明显.
