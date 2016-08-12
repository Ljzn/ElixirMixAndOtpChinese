#分布式任务与配置

  1. 我们的第一个分布式代码
  2. 同步/等待
  3. 分布式任务
  4. 路由层
  5. 测试过滤器与标签
  6. 应用环境与配置
  7. 总结

本章,我们将回到`:kv`应用并添加一个路由层,它能让我们根据桶名来在节点之间分布请求.

路由层将会以如下格式收到一个路径表格:

```
[{?a..?m, :"foo@computer-name"},
 {?n..?z, :"bar@computer-name"}]
```

路由器将会检查桶名的第一个字节是否在表格中,并依此将其派遣到合适的节点.例如,一个以字母"a"(`?a`代表字母a的Unicode代码点)开头的桶将会被派遣到`foo@computer-name`节点.

如果匹配到的入口指向了能评估该请求的节点,那么我们就完成了寻路,然后这个节点将会执行所请求的操作.如果匹配到的入口指向了不同的节点,我们将传送请求到该节点,它会搜寻自己的路由表格(也许会和第一个节点中的不同)并做出相应的动作.如果没有入口被匹配到,将会抛出一个错误.

你可能会想知道为什么我们不直接让我们在路由表格中找到的节点去执行所请求的操作,而是将路由请求传递到该节点来处理.因为像上面这种简单的路由表格可以合理的被所有节点分享,以这种方式传递路由请求能够很容易地在我们的应用成长时将路径表格分解成更小的块.也许是同一个原因,`foo@computer-name`将只会对路由桶请求负责,它处理的桶将会被派遣到不同的节点.这样,`bar@computer-name`就不需要知道这些变化.

> 注意:本章中我们将在同一个机器上使用两个节点.你可以在同个网络下使用两个或更多不同的机器,但是你需要做一些准备工作.首先,你需要确认所有机器都有一个有着相同值得`~/.erlang.cookie`文件.其次,你需要保证epmd运行在一个未阻塞的端口(你可以运行`epmd -d`获取调式信息).然后,如果你想学习更多关于分布的内容,我们推荐[Learn You Some Erlang中的Distribunomicon]("http://learnyousomeerlang.com/distribunomicon")这一章.

#我们第一个分布式代码

Elixir装载了许多用来连接节点并交换信息的工具.事实上,我们使用了和进程相同的概念,能够在分布式环境中发送和接受信息,是因为Elixir进程是_位置透明_的.意思是当发送信息时,收件人是否在同一个节点不重要,VM在这两种情形下都能够传送信息.

为了运行分布式代码,我们需要启动一个具名VM.名字可短(当在同一个网络)可长(需要完整的电脑地址).让我们开启一个新的IEx会话:

```
$ iex --sname foo
```

你会发现提示符有些不同,它显示了节点名称,后面跟着电脑名称:

```
Interactive Elixir - press Ctrl+C to exit (type h() ENTER for help)
iex(foo@jv)1>
```

我的电脑名是`jv`,所以我在上面的例子中看到的是`foo@jv`,但你的会不同.我们将在下面的例子中使用`foo@computer-name`,当输入这些代码时你需要按情况更新.

让我们在这个壳中定义一个名为`Hello`的模块:

```
iex> defmodule Hello do
...>   def world, do: IO.puts "hello world"
...> end
```

如果在同一个网络中你有另一台安装了Erlang和Elixir的电脑,你可以在上面启动另一个壳.如果没有,你可以简单地在另一个终端中启动另一个IEx会话.同样地,给它一个短名叫做`bar`:

```
$ iex --sname bar
```

注意在这个新的IEx会话中,我们不能访问`Hello.world/0`:

```
iex> Hello.world
** (UndefinedFunctionError) undefined function: Hello.world/0
    Hello.world()
```

然而我们可以在`bar@computer-name`上为`foo@computer-name`生成一个新的进程!让我们来试一试:

```
iex> Node.spawn_link :"foo@computer-name", fn -> Hello.world end
#PID<9014.59.0>
hello world
```

Elixir在另一个节点生成了一个进程并返回了它的pid.然后代码在`Hello.world/0`存在的节点执行,并调用那个函数.注意其结果`hello world`打印在当前节点`bar`上,而不是`foo`.换句话说,被打印的信息是从`foo`发送到了`bar`.这是因为在另一个节点(`foo`)生成的进程仍然有一个当前节点(`bar`)的群首领.我们曾在IO章节中简短地讨论过群首领.

我们可以像往常一样使用由`Node.spawn_link/2`返回的pid来收发信息.让我们来尝试一个快速乒乓的例子:

```
iex> pid = Node.spawn_link :"foo@computer-name", fn ->
...>   receive do
...>     {:ping, client} -> send client, :pong
...>   end
...> end
#PID<9014.59.0>
iex> send pid, {:ping, self}
{:ping, #PID<0.73.0>}
iex> flush
:pong
:ok
```

由此,我们可以得出结论,当我们需要进行分布式计算时,我们应该使用`Node.spawn_link/2`来在远程节点生成进程.然而,在本教程中我们已经学过,应当尽量避免在监督树之外生成进程,所以我们需要寻找其它选项.

这里有三个能在我们的实现中使用的,`Node.spawn_link/2`的替代品:

  \1. 我们可以使用Erlang的:rpc模块来在远程节点执行函数.在`bar@computer-name`壳中,你可以调用`:rpc.call(:"foo@computer-name", Hello, :world, [])`,然后它将打印"hello world"

  \2. 我们可以有一个运行在其它节点上的服务器,并通过GenServer API向该节点发送请求.例如,你可以使用`GenServer.call({name, node}, arg)`来调用一个远程具名服务器,或者简单地将远程进程的PID作为第一个参数传送.

  \3. 我们可以使用在上一章中所学到的tasks,因为它们在本地和远程节点都可以被生成.

上述选项有着不同的特性.`:rpc`和使用GenServer都会在一个服务器上将你的请求序列化,而tasks可以高效地同步运行在远程节点,并由主管来生成序列点.

对于我们的路由层,我们将使用tasks,但你可以自由地探索其它替代品.

#同步/等待(async/await)

目前我们已经探索了独立启动和运行的tasks,不考虑它们的返回值.然而,运行一个task来计算一个值并读取它的结果有时是很有用的.所以,tasks也提供了`async/await`模式:

```
task = Task.async(fn -> compute_something_expensive end)
res  = compute_something_else()
res + Task.await(task)
```

`async/await`提供了一个非常简单的机制来同时计算值.不仅如此,`async/await`还可用于我们在上一章中提到的`Task.Supervisor`.我们只需要用`Task.Supervisor.async/2`替代`Task.Supervisor.start_child/2`,并使用`Task.await/2`在稍后读取结果.

#分布式任务(Distributed tasks)

分布式任务和受监督任务几乎完全一样.唯一的不同点是当我们在主管上生成task时,我们传送的是节点名.打开`:kv`应用中的`lib/kv/supervisor.ex`.让我们添加一个task主管,作为树的最后一个孩子:

```
supervisor(Task.Supervisor, [[name: KV.RouterTasks]]),
```

现在,让我们再次启动两个具名节点,但是在`:kv`应用中:

```
$ iex --sname foo -S mix
$ iex --sname bar -S mix
```

在`bar@computer-name`之中,我们现在可以利用主管直接生成一个其它节点内的task:

```
iex> task = Task.Supervisor.async {KV.RouterTasks, :"foo@computer-name"}, fn ->
...>   {:ok, node()}
...> end
%Task{pid: #PID<12467.88.0>, ref: #Reference<0.0.0.400>}
iex> Task.await(task)
{:ok, :"foo@computer-name"}
```

我们的第一个分布式task简单地检索了正在运行的节点名.注意,我们给了`Task.Supervisor.async/2`一个匿名函数,但是在分布式的情况下,更推荐明确地给定模块,函数和参数:

```
iex> task = Task.Supervisor.async {KV.RouterTasks, :"foo@computer-name"}, Kernel, :node, []
%Task{pid: #PID<12467.88.0>, ref: #Reference<0.0.0.400>}
iex> Task.await(task)
:"foo@computer-name"
```

区别在于匿名函数要求目标节点的代码版本要和调用者完全一样.使用模块,函数和参数会更健壮,因为你只需要在给定的模块中找到一个能够匹配参数的函数.

利用已学的知识,让我们来编写路由代码吧.

#路由层(Routing layer)

创建`lib/kv/router.ex`文件:

```
defmodule KV.Router do
  @doc """
  Dispatch the given `mod`, `fun`, `args` request
  to the appropriate node based on the `bucket`.
  """
  def route(bucket, mod, fun, args) do
    # Get the first byte of the binary
    first = :binary.first(bucket)

    # Try to find an entry in the table or raise
    entry =
      Enum.find(table, fn {enum, _node} ->
        first in enum
      end) || no_entry_error(bucket)

    # If the entry node is the current node
    if elem(entry, 1) == node() do
      apply(mod, fun, args)
    else
      {KV.RouterTasks, elem(entry, 1)}
      |> Task.Supervisor.async(KV.Router, :route, [bucket, mod, fun, args])
      |> Task.await()
    end
  end

  defp no_entry_error(bucket) do
    raise "could not find entry for #{inspect bucket} in table #{inspect table}"
  end

  @doc """
  The routing table.
  """
  def table do
    # Replace computer-name with your local machine name.
    [{?a..?m, :"foo@computer-name"},
     {?n..?z, :"bar@computer-name"}]
  end
end
```

让我们编写一个测试来验证路由器的工作.创建一个名为`test/kv/router_test.exs`的文件:

```
defmodule KV.RouterTest do
  use ExUnit.Case, async: true

  test "route requests across nodes" do
    assert KV.Router.route("hello", Kernel, :node, []) ==
           :"foo@computer-name"
    assert KV.Router.route("world", Kernel, :node, []) ==
           :"bar@computer-name"
  end

  test "raises on unknown entries" do
    assert_raise RuntimeError, ~r/could not find entry/, fn ->
      KV.Router.route(<<0>>, Kernel, :node, [])
    end
  end
end
```

第一个测试简单地调用了`Kernel.node/0`,它会基于桶名"hello"和"world"来返回当前节点的名字.依据我们的路由表格,我们应当会分别得到`foo@computer-name`和`bar@computer-name`作为回复.

第二个测试只是检查对于未知入口的报错.

为了运行第一个测试,我们需要运行两个节点.进入`apps/kv`,并重启节点`bar`.

```
$ iex --sname bar -S mix
```

以如下命令运行测试:

```
$ elixir --sname foo -S mix test
```

我们将成功通过测试.优秀!

#测试过滤器与标签

尽管我们的测试通过了,我们的测试结构却变得更复杂了.特别地,使用`mix test`运行测试将导致失败,因为我们的测试要求连接到另一个节点.

幸运的是,ExUnit装载了测试标签的功能,让我们能运行特定的回调或者基于那些标签来过滤测试.在之前的章节我们已经使用了`:captrue_log`标签,它是由ExUnit自己定义的.

这一次让我们添加一个`:distributed`标签到`test/kv/router_test.exs`:

```
@tag :distributed
test "route requests across nodes" do
```

`@tag :distributed`等同于`@tag distributed: true`.

当测试被合适地标上标签后,我们可以检查网络上的节点是否活着,如果没有,我们可以排除所有分布式测试.打开`:kv`应用中的`test/test_helper.exs`,并添加:

```
exclude =
  if Node.alive?, do: [], else: [distributed: true]

ExUnit.start(exclude: exclude)
```

现在,用`mix test`运行测试:

```
$ mix test
Excluding tags: [distributed: true]

.......

Finished in 0.1 seconds (0.1s on load, 0.01s on tests)
7 tests, 0 failures, 1 skipped
```

这一次所有的测试都通过了,而且ExUnit警告我们分布式测试被排除了.如果你使用`$ elixir --sname foo -S mix test`运行测试,另一个额外的测试就会执行并成功通过,只要`bar@computer-name`节点可用.

`mix test`命令也允许我们动态地包含和排除标签.例如,我们可以运行`$ mix test --include distributed`来运行分布式测试,不管`test/test_helper.exs`中的设置是怎样.我们也可以传送`--exclude`来排除特定的标签.最后,`--only`可以被用于只运行特定标签的测试:

```
$ elixir --sname foo -S mix test --only distributed
```

#应用环境与配置

现在我们是直接在代码中将路由表格写入`KV.Router`模块.然而,我们希望将表格变为动态的.这使得我们不单单要配置开发/测试/生产模式,还要让不同的节点使用不同入口的路由表格.这就是OTP的特性之一:应用环境.

每个应用都有一个环境,其中用键存储了应用的特定配置.例如,我们可以将路由表格存储在`:kv`应用的环境中,给它一个默认值,并让其它应用按需修改表格.

打开`apps/kv/mix.exs`并修改`application/0`函数:

```
def application do
  [applications: [],
   env: [routing_table: []],
   mod: {KV, []}]
end
```

我们添加了一个新的`:env`键到应用中.它返回的是应用的默认环境,它有一个入口键`:routing_table`和一个空列表作为值.应用环境中装载一个空表格是有意义的,因为特定的路由表格依赖于测试/部署的结构.

为了在我们的代码中使用应用环境,我们只需要修改`KV.Router.table/0`的定义:

```
@doc """
The routing table.
"""
def table do
  Application.fetch_env!(:kv, :routing_table)
end
```

我们使用`Application.fetch_env!/2`来从`:kv`环境中的`:routing_table`里读取入口.

由于我们的路由表格目前是空的,我们的分布式测试将会失败.重启应用并再次运行测试:

```
$ iex --sname bar -S mix
$ elixir --sname foo -S mix test --only distributed
```

关于应用环境的一件有趣的事情是它不仅是为当前应用配置,而是为所有应用.这些配置是由`config/config.exs`文件完成的.例如,我们可以配置IEx的默认提示符.只需要打开`apps/kv/config/config.exs`并添加如下内容到末尾:

```
config :iex, default_prompt: ">>>"
```

使用`iex -S mix`来启动IEx,你会发现IEx的提示符改变了.

这意味着我们也可以在`apps/kv/config/config.exs`文件中直接配置我们的`:routing_table`:

```
# Replace computer-name with your local machine nodes.
config :kv, :routing_table,
       [{?a..?m, :"foo@computer-name"},
        {?n..?z, :"bar@computer-name"}]
```

重启节点并再次运行分布式测试.现在它们都通过了.

从Elixir v1.2开始,所有雨伞应用会共用它们的配置,多亏了雨伞根目录中`config/config.exs`文件中的这行代码,载入了所有孩子的配置:

```
import_config "../apps/*/config/config.exs"
```

`mix run`命令也接受一个`--config`旗帜,它允许我们按需提供配置文件.这可以用于开启不同的节点,每个有着它自己的配置(例如,不同的路由表格).

内置的应用配置能力和雨伞应用的结构给了我们很多的选项,在部署我们的软件时.我们能:

  \- 部署一个雨伞应用到一个节点,它既是一个TCP服务器,又是一个键值存储器

  \- 部署一个`:kv_server`应用,只要路由表格只指向其它节点,它就只作为一个TCP服务器

  \- 部署一个`:kv`应用,让一个节点只作为存储器(没有TCP入口)

我们将在未来添加更多的应用,我们可以继续以相同的粒度控制我们的部署,应用与配置的最佳选择都取决于产品.

你也可以考虑使用像exrm这样的工具来构建多个版本,它将封装你所选择的应用和配置,包括当前安装的的Erlang和Elixir,所以我们可以在没有预先安装好runtime的目标系统上部署应用.

最后,本章我们已经学习了一些新的东西,它们可以被应用于`:kv_server`.以下的步骤将作为练习:

  \- 使`:kv_server`应用从它的应用环境中读取端口,而不是使用硬代码的4040值

  \- 让`:kv_server`应用使用路由功能,替代直接分发到本地的`KV.Registry`.在`:kv_server`测试中,你可以让路由表格简单地指向当前节点

#总结

本章我们构建了一个简单的路由器,作为探索Elixir和Erlang VM的分布式特性的方法,还学习了如何配置它的路由表格.这是我们的Mix和OTP教程的最后一章.

在整个教程中,我们构建了一个非常简单的分布式键值存储,作为一个探索各种结构的机会,例如通用服务器,主管,任务,代理,应用等等.不仅如此,我们还为整个应用编写了测试,熟悉了ExUnit,还学习了如何使用Mix构建工具来完成大范围的任务.

如果你正在寻找一个能在生产中使用的分布式键值存储,你绝对应该考虑Riak,它也运行在Erlang VM上.在Riak中,桶是可复制的,为了避免数据丢失,他们使用了一致性散列来将桶映射到节点上,而不是使用路由.一致性散列算法有助于减少需要迁移的数据,当新的用来存储桶的节点被添加到你的基础设施中时.

这里还有更多的课程要学习,我们希望你学的开心!
