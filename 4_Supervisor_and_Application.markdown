#主管与应用

  1. 我们的第一个主管
  2. 理解应用
    1. 开启应用
    2. 应用回调
    3. 项目还是应用?
  3. 简单的一对一主管
  4. 监督树
  5. 观察者
  6. 测试中的共用状态

现在,我们的应用有个一个能监控几十个桶,不是几百个,的注册表.尽管我们认为目前的实现很不错,但任何软件都会有bug,失败也肯定会发生.

当时事件失败时,你的第一反应可能是:"让我们来挽救这些错误吧".但在Elixir中我们要避免这种在其它语言中常见的抢救异常的防御式编程习惯.相反,我们说:"让它崩溃".如果有一个bug导致我们的注册表崩溃,我们没有什么好担心的,因为我们要设置一个主管,它会开启一个新的注册表副本.

本章,我们将学习主管和应用.我们要创造的不是一个,而是两个主管,并用它们来监督我们的进程.

#我们的第一个主管

创建一个主管与创建一个GenServer差不多.我们将在`lib/kv/supervisor.ex`文件中定义一个名为`KV.Supervisor`的模块,它将行使主管的行为:

```
defmodule KV.Supervisor do
  use Supervisor

  def start_link do
    Supervisor.start_link(__MODULE__, :ok)
  end

  def init(:ok) do
    children = [
      worker(KV.Registry, [KV.Registry])
    ]

    supervise(children, strategy: :one_for_one)
  end
end
```

现在我们的主管有一个单独的孩子:注册表.一个工人的格式:

```
worker(KV.Registry, [KV.Registry])
```

将使用以下调用来开始一个进程:

```
KV.Registry.start_link(KV.Registry)
```

我们传送给`start_link`的参数是进程的名字.为被监督的进程命名是很常见的,这样其它进程就可用通过名称来访问它们,而不需要知道它们的pid.这很有用,因为被监督的进程有可能崩溃,在主管将其重启后pid会改变.通过命名,我们可以保证新启动的进程将用同一个名称注册自己,而不需要明确获取最新的pid.注意将进程名设为定义它的模块名
也是很常见的,这使得在一个活系统中进行调试或检测都变得更加直观.

最后,我们调用了`supervise/2`,传送了孩子的列表,以及`:one_for_one`策略.

监督策略决定了当一个孩子崩溃时会发生什么.`:one_for_one`意味着如果一个孩子死了,将会只有一个重新启动.因为我们目前只有一个孩子,所以这就是我们需要的.`Supervisor`行为支持许多不同的策略,我们将在本章中讨论他们.

因为`KV.Registry.start_link/1`现在需要一个参数,所以我们要改变我们的实现来接收这个参数.打开`lib/kv/registry.ex`并将`start_link/0`的定义改为:

```
@doc """
 Starts the registry with the given `name`.
 """
 def start_link(name) do
   GenServer.start_link(__MODULE__, :ok, name: name)
 end
```

我们还需要更新我们的测试,一遍在开启注册表时提供一个名字.将`test/kv/registry_test.exs`中的`setup`函数修改为:

```
setup context do
  {:ok, registry} =  KV.Registry.start_link(context.test)
  {:ok, registry: registry}
end
```

`setup/2`也可以接收测试内容,类似于`test/3`.除了任何我们想要添加进设置块中的值,内容还包括了一些默认的键,例如`:case`,`:test`,`file`和`line`.我们使用`context.test`作为快捷方式来生成一个与当前运行的测试同名的注册表.

随着我们的测试通过,现在我们可以让主管运做起来.如果我们在项目内用`iex -S mix`启动一个控制台,我们可以手动启动主管:

```
iex> KV.Supervisor.start_link
{:ok, #PID<0.66.0>}
iex> KV.Registry.create(KV.Registry, "shopping")
:ok
iex> KV.Registry.lookup(KV.Registry, "shopping")
{:ok, #PID<0.70.0>}
```

当我们启动了主管,注册表工人就自动启动了,允许我们创建桶,而不需要手动启动它.

在实际中,我们很少手动启动应用主管.相反,它是作为程序回调的一部分来启动的.

#理解应用

我们始终是在一个应用里工作.每当我们修改了一个文件并运行mix编译时,我们会在编译输出中看到一个生成kv应用的消息.

我们可以在`_build/dev/lib/kv/ebin/kv.app`文件中找到生成了的`.app`文件.让我们看看它的内容:

```
{application,kv,
             [{registered,[]},
              {description,"kv"},
              {applications,[kernel,stdlib,elixir,logger]},
              {vsn,"0.0.1"},
              {modules,['Elixir.KV','Elixir.KV.Bucket',
                        'Elixir.KV.Registry','Elixir.KV.Supervisor']}]}.
```

该文件包含了Erlang术语(用Erlang语法).即使我们不熟悉Erlang,也很容易猜出这个文件保存的是我们的应用定义.它包含了我们的应用版本,所有模块定义,还有我们依赖的应用,例如Erlang的`kernel`,`elixir`本身,还有`mix.exs`中的应用列表指定的`logger`.

每添加一个新的模块到我们的应用,就要手动更新这个文件,那会非常无聊.这就是为什么Mix会为我们生成和维护它.

我们也可以通过自定义`mix.exs`项目文件中的`application/0`的返回值来配置生成的`.app`文件.我们马上将制作我们的第一个定制文件.

##开启应用

当我们定义了应用的规范,即`.app`文件之后,我们就能够将应用作为一个整体来开关.目前我们还不用为此而担心,因为:

  1. Mix为我们自动开启了当前应用
  2. 即使Mix没有为我们开启应用,我们的应用在启动时也不会做任何事

不管怎样,让我们看看Mix是如何为我们开启应用的.让我们使用`iex -S mix`来启动一个项目控制台,并试着运行:

```
iex> Application.start(:kv)
{:error, {:already_started, :kv}}
```

噢,它已经启动了.Mix通常会启动在我们项目的`mix.exs`文件中定义的应用整体结构,并且同样对待所有的依赖,如果有起来其它应用的话.

我们可以传送一个选项给Mix,告诉它不要启动我们的应用.试着运行`iex -S mix run --no-start`:

```
iex> Application.start(:kv)
:ok
```

我们可以停止`:kv`应用,以及`:logger`应用,它是由Elixir默认启动的:

```
iex> Application.stop(:kv)
:ok
iex> Application.stop(:logger)
:ok
```

让我们再次启动我们的应用:

```
iex> Application.start(:kv)
{:error, {:not_started, :logger}}
```

现在我们得到了一个错误,因为应用`:kv`的依赖(这时是`:logger`)没有启动.我们需要手动地以正确顺序启动每个应用,或者调用`Application.ensure_all_started`:

```
iex> Application.ensure_all_started(:kv)
{:ok, [:logger, :kv]}
```

没有什么exciting的事情发生,但展示了如何控制我们的应用.

> 当你运行`iex -S mix`时,相当于运行`iex -S mix run`.所以无论何时当你需要在启动IEx时传送更多的选项给Mix,就只需要输入`iex -S mix run`并传送任何`run`命令可以接受的选项.你可以找到关于`run`的更多信息,通过在你的壳中运行`mix help run`.

##应用回调

我们把所有时间花在了如何使应用开始和停止上,现在我们必须在应用开始后让它做一些有用的事情.

我们可以指定一个应用回调函数.这是一个将在应用启动时调用的函数.函数必须返回`{:ok, pid}`,这里的`pid`是监督树进程的标识符.

两步配置应用回调.首先打开`mix.exs`文件,修改`def application`:

```
def application do
    [applications: [:logger],
     mod: {KV, []}]
end
```

`:mod`选项指定了"应用回调模块",通过在应用程序启动时传递的参数.应用回调模块可以是实现应用行为的任何模块.

现在我们已经指定`KV`为模块回调,我们需要修改`KV`模块在`lib/kv.ex`中的定义:

```
defmodule KV do
  use Application

  def start(_type, _args) do
    KV.Supervisor.start_link
  end
end
```

当`use Application`时,我们需要定义一些函数,类似于我们使用`Supervisor`或`GenServer`时.这次我们只需要定义一个`start/2`函数.如果我们想定义应用停止时的行为,我们可以定义`stop/2`函数.

让我们再次运行`iex -S mix`启动我们的项目控制台.我们会看见一个名为`KV.Registry`的进程已经在运行了:

```
iex> KV.Registry.create(KV.Registry, "shopping")
:ok
iex> KV.Registry.lookup(KV.Registry, "shopping")
{:ok, #PID<0.88.0>}
```

我们是如何知道它正在工作的?毕竟,我们创建了桶,并查看了它;它当然在工作,对吗?好吧,记住`KV.Registry.create/2`使用`GenServer.cast/3`,因此无论消息是否找到了它的目标,都会返回`:ok`.这一点上,我们不知道主管和服务器是否已经启动,桶是否已经被创建.然而,`KV.Registry.lookup/2`使用`GenServer.call/3`,它会阻塞并等待服务器响应.我们得到了积极反应,所以我们知道一切都已经启动并运行了.

做个试验,试着用`GenServer.call/3`重新实现`KV.Registry.create/2`,并暂时禁用应用回调.在控制台中再次运行以上代码,你会看到在创建阶段就直接失败了.

别忘了在继续本教程之前将代码改回来!

##项目还是应用?

Mix对于项目和应用是区分对待的.基于`mix.exs`文件的内容,我们会说我们有一个定义了`:kv`应用的Mix项目.正如我们将在后面的章节中看到的,有一些项目不定义任何应用.

当我们说"项目"时,你应当想到Mix.Mix是管理你的项目的工具.它知道如何编译,测试你的项目等等.它也知道如何编译和启动与你的项目相关的应用.

当我们谈论应用时,我们是在谈论OTP.应用是由启动,运行和停止组合成的实体.你可以在应用模块的文档中了解更多关于应用的信息,还可通过运行`mix help compile.app`来学习更多`def application`支持的选项.

#简单的一对一主管

我们已经成功定义了我们的主管,它是作为应用生命周期的一部分自动启动(及停止)的.

但是要记住,在`handle_cast/2`回调中,`KV.Registry`既链接了又监控了桶进程:

```
{:ok, pid} = KV.Bucket.start_link
ref = Process.monitor(pid)
```

链接是双向的,这意味桶的崩溃会导致注册表崩溃.虽然我们现在有了主管,这保证了注册表将备份和运行,注册表的崩溃仍意味着我们将失去所有的桶名与进程间对应关系的数据.

换句话说,我们希望即使一个桶崩溃,注册表也保持运行.让我们来写一个新的注册表测试:

```
test "removes bucket on crash", %{registry: registry} do
  KV.Registry.create(registry, "shopping")
  {:ok, bucket} = KV.Registry.lookup(registry, "shopping")

  # 因特殊原因停止桶
  Process.exit(bucket, :shutdown)

  # 等待直到桶死了
  ref = Process.monitor(bucket)
  assert_receive {:DOWN, ^ref, _, _, _}

  assert KV.Registry.lookup(registry, "shopping") == :error
end
```

测试类似于"在退出时删除桶",除了我们将发送`:shutdown`替代`:normal`作为退出的理由.与`Agent.stop/1`相反,`Process.exit/2`是一个异步操作,因此我们不能简单地在发送退出信号后查询`KV.Registry.lookup/2`,因为不能保证桶会立刻死亡.为了解决这个问题,我们还在测试过程中监视桶,并只在我们确认桶关闭了,才查询注册表,避免竞争条件.

因为桶链接到了注册表,而注册表链接到了测试进程,杀死桶将导致注册表崩溃,而后导致测试进程崩溃:

```
1) test removes bucket on crash (KV.RegistryTest)
   test/kv/registry_test.exs:52
   ** (EXIT from #PID<0.94.0>) shutdown
```

一种可能的解决方案是提供一个`KV.Bucket.start/0`,它调用了`Agent.start/1`,并在注册表中使用,消除了注册表与桶的链接.然而,这是个坏主意,因为桶将不会被链接到任何进程.这意味着如果有人停止了`:kv`应用,这些桶仍然活着,因为它们是孤立的.
不仅如此,如果一个进程是孤立的,它就更难内省.

我们将通过定义一个新的主管来解决这个问题,它会生成并监督所有的桶.这里有一种监督策略,叫做`:simple_one_for_one`,很适合这种情形:我们可以选定一个工人模板,让后依照这个模板管理很多孩子.使用这个策略后,在主管初始化时,没有工人被启动,新的工人会在每次调用`start_child/2`时启动.

 让我们这样定义`lib/kv/bucket/supervisor.ex`中的`KV.Bucket.Supervisor`:

 ```
 defmodule KV.Bucket.Supervisor do
  use Supervisor

  # 一个简单的模块属性用来存储主管名
  @name KV.Bucket.Supervisor

  def start_link do
    Supervisor.start_link(__MODULE__, :ok, name: @name)
  end

  def start_bucket do
    Supervisor.start_child(@name, [])
  end

  def init(:ok) do
    children = [
      worker(KV.Bucket, [], restart: :temporary)
    ]

    supervise(children, strategy: :simple_one_for_one)
  end
end
```

相较于之前的主管有了三个变化.

不再将注册进程名当做参数来接收,我们直接将其命名为`KV.Bucket.Supervisor`,因为我们不想生成该进程的其它版本.我们也定义了一个`start_bucket/0`函数,它会开启一个桶,作为我们的主管的孩子,名字是`KV.Bucket.Supervisor`.`start_bucket/0`是一个我们将要调用的函数,用来替代在这册表中直接调用`KV.Bucket.start_link`.

最后,在`init/1`回调中,我们将工人标记为`:temporary`.这意味着如果桶死了,将不会重启!这是因为我们只想用主管作为一个联合桶的机制.桶的创建将总是经过注册表.

运行`iex -S mix`,试用一下我们的新主管:

```
iex> {:ok, _} = KV.Bucket.Supervisor.start_link
{:ok, #PID<0.70.0>}
iex> {:ok, bucket} = KV.Bucket.Supervisor.start_bucket
{:ok, #PID<0.72.0>}
iex> KV.Bucket.put(bucket, "eggs", 3)
:ok
iex> KV.Bucket.get(bucket, "eggs")
3
```

让我们重写桶的创建方法,来改变注册表和桶主管的工作方式:

```
def handle_cast({:create, name}, {names, refs}) do
  if Map.has_key?(names, name) do
    {:noreply, {names, refs}}
  else
    {:ok, pid} = KV.Bucket.Supervisor.start_link
    ref = Process.monitor(pid)
    refs = Map.put(refs, ref, name)
    names = Map.put(names, name, pid)
    {:noreply, {names, refs}}
  end
end
```

一旦完成了这些更改,我们的测试应该会失败,因为这里没有桶主管.让我们自动地开启桶主管,作为我们监督树的一部分,而不是直接在每个测试中启动桶主管.

#监督树

为了在我们的应用中使用桶主管,我们需要将其作为一个孩子添加到`KV.Supervisor`.注意我们有一个用来监督其它主管的主管,这种结构叫"监督树".

打开`lib/kv/supervisor.ex`,并修改`init/1`:

```
def init(:ok) do
  children = [
    worker(KV.Registry, [KV.Registry]),
    supervisor(KV.Bucket.Supervisor, [])
  ]

  supervise(children, strategy: :one_for_one)
end
```

我们已经将一个主管当成孩子添加进去了,并且会不带参数地启动它.重新运行测试,所有测试都将通过.

因为我们添加了一个孩子到主管,所以有必要确认`:one_for_one`主管策略是否还奏效.出现了一个缺陷,就是`KV.Registry`工人进程与`KV.Bucket.Supervisor`主管进程的关系.如果`KV.Registry`死了,所有`KV.Bucket`名与`KV.Bucket`进程的联系信息都会丢失,因此`KV.Bucket.Supervisor`也必须死亡--否则,它管理的`KV.Bucket`进程将被孤立.

经过观察,我们决定转换主管策略.有两个候选的是`:one_for_all`和`rest_for_one`.一个使用`:one_for_all`策略的主管,将会杀死并重启它所有的子进程,在其中任何一个死亡时.第一眼看上去似乎很适合我们,但它有些太笨拙,因为当`KV.Bucket.Supervisor`死亡时,`KV.Registry`能够很完美地清理自己.这时,`:rest_for_one`策略就很合适了,当一个子进程崩溃时,主管只会杀死并重启那些在崩溃的进程之后启动的子进程.让我们重写监督树来改用这个策略:

```
def init(:ok) do
  children = [
    worker(KV.Registry, [KV.Registry]),
    supervisor(KV.Bucket.Supervisor, [])
  ]

  supervise(children, strategy: :rest_for_one)
end
```

现在,如果注册表工人崩溃了,注册表和`KV.Supervisor`"其余的"的孩子(例如`KV.Bucket.Supervisor`)将会重启.然而,如果`KV.Bucket.Supervisor`崩溃了,`KV.Registry`将不会重启,因为它是在`KV.Bucket.Supervisor`之前开启的.

这里还有许多其它策略和选项可用于`worker/2`,`supervisor/2`和`supervise/2`函数,别忘了查看`Supervisor`和`Supervisor.Spec`模块.

距离下一章还有两小节.

#观察器

现在我们定义好了监督树,这是一个很好的机会来介绍Erlang搭载的观察者工具.运行`iex -S mix`并输入:

```
iex> :observer.start
```

一个包含了我们的系统的所有信息的用户界面将会出现,从通用静态资源表,到正在运行的进程和应用列表.

在Application那一栏,你可以看到系统中正在运行的应用的监督树.你可以选择`kv`应用来进一步查看.

不仅如此,当你在终端中创建新桶,在观察器中可以看到其在监督树上生成的新进程:

```
iex> KV.Registry.create KV.Registry, "shopping"
:ok
```

我们将让你自己探索观察器的其它部分.注意你可以双击监督树中的任何进程来获取它的更多信息,还可以右击来发送"死亡信号",这是一个制造失败的完美方式,以此来观察主管的反应是否和预期一样.

最后,你总是在监督树中启动进程的主要原因之一,就是这些类似观察器的工具能够确保进程是可访问和可内省的,即使它们是暂时的.

#测试中的共用状态

之前我们为每个测试启动一个注册表,来确保它们是独立的:

```
setup context do
  {:ok, registry} = KV.Registry.start_link(context.test)
  {:ok, registry: registry}
end
```

现在我们已经将注册表改用了`KV.Bucket.Supervisor`,它是全局注册表,我们的测试现在依赖于这个共用的全局主管,即使每个测试拥有它自己的注册表.问题是:我们应当这样做吗?

看情况.我们可以依赖于共用全局状态,只要我们依赖于该状态的不共用部分.例如,每次我们注册一个进程到给定名称下,我们都注册了一个进程给共享名称注册表.然而,只要我们保证这些名称是不同的,通过使用类似于`contest.test`的结构,我们就不会有并发或数据依赖上的问题.

相同的原因也适用于我们的桶主管.尽管多重注册可能会在共用的桶主管上启动桶,那些桶和注册表都是独立的.唯一我们可能遇到的并发问题就是如果我们调用类似于`Supervisor.count_children(KV.Bucket.Supervisor)`的函数,它将会计算所有的注册表中的所有桶,当测试并发运行时,可能会得到不同的结果.

由于我们目前依赖于桶主管不共用的部分,所以我们不用担心测试中的并发问题.当这成为问题时,我们可以为每个测试启动一个主管,并将其作为一个参数传递给注册表`start_link`函数.

现在我们的应用已经合适地被监督和测试过了,让我们看看如何让它更快.
