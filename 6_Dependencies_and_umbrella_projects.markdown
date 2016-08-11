#依赖和雨伞项目

  1. 外部依赖
  2. 内部依赖
  3. 雨伞项目
  4. 伞内依赖
  5. 总结

本章,我们将讨论如何在Mix中管理依赖.

我们的`kv`应用已经完成,所以是时候实现能够处理我们在第一章中定义的请求的服务器了:

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

然而,我们将构建一个最为另一个应用的TCP服务器,它是`kv`应用的一个客户端.而不是添加更多代码到`kv`应用中.因为对于应用来说,整个运行时和Elixir生态系统都是它的齿轮,所以我们能够把我们的项目分解成多个更小的项目,而不是组建一个巨大的,整个的应用.

在创建应用之前,我们必须讨论Mix是如何处理依赖的.在实际中,有两种我们常见的依赖:内部和外部依赖.Mix支持这两种机制.

#外部依赖

外部依赖就是你的业务范围之外的东西.例如,如果你需要一个HTTP API给你的分布式KV应用,你可以使用Plug项目作为一个外部依赖.

安装外部依赖很简单.通常,我们使用Hex包管理工具,它能列出在我们的`mix.exs`文件中所有deps函数之内的依赖.

```
def deps do
  [{:plug, "~> 1.0"}]
end
```

这个依赖的定义是指Plug 1.x.x系列的最新版本已经被Hex添加了.这是由版本号之前的`~>`表明的.想知道更多关于制定版本要求的信息,请查阅Version模块的文档.

一般,稳定的版本会被加入Hex.如果你想要依赖于一个仍在开发阶段的外部依赖,Mix也能够管理git依赖:

```
def deps do
  [{:plug, git: "git://github.com/elixir-lang/plug.git"}]
end
```

你会发现当你添加了一个依赖到你的项目,Mix生成了一个`mix.lock`文件,它保证了构建的可重复性.锁文件必须被导入到你的版本控制系统中,来保证每个人使用该项目时,依赖的版本与你相同.

Mix提供了许多命令来处理依赖,可以再`mix help`中看到:

```
$ mix help
mix deps              # 列出依赖,和它们的状态
mix deps.clean        # 删除给定的依赖的文件
mix deps.compile      # 编译依赖
mix deps.get          # 获取所有最新版本的依赖
mix deps.unlock       # 解锁给定的依赖
mix deps.update       # 更新给定的依赖
```

最常用的命令是`mix deps.get`与`mix deps.update`.获取依赖之后,它会自动编译.你可以通过`mix help deps`或查看Mix.Task.Deps模块的文档,来获取更多关于deps的信息.

#内部依赖

内部依赖是特定于你的项目的.它们通常在你的项目/公司/机构之外就没有意义.多数时候,出于技术,经济还是业务上的原因,你想要保持它们的私用性.

如果你有一个内部依赖,Mix支持两种操作方法:git仓库或雨伞计划.

例如,如果你添加`kv`项目到一个git仓库,你只需要在你的deps代码中按顺序列出它们就能使用:

```
def deps do
  [{:kv, git: "https://github.com/YOUR_ACCOUNT/kv.git"}]
end
```

如果仓库是私有的,你可能需要指定一个私有URL`git@github.com:YOUR_ACCOUNT/kv.git`.任何时候,Mix都会获取到它,只要你有合适的凭证.

在Elixir中不是特别推荐使用git依赖.记住运行时和Elixir生态系统已经提供了应用的概念.所以,我们希望你能经常地将你的代码打碎成多个应用,它们能够在本地组合,即使是在单个项目中.

然而,如果将每个应用作为独立的项目添加到git仓库,你的项目会变得很难维护,因为你会花费大量时间来管理这些git仓库,而不是写你的代码.

出于该原因,Mix支持了"雨伞计划".雨伞计划允许你创建一个包含着许多应用的项目,它们全部都放在一个单个的源代码仓库里.这就是下一部分我们将探索的方法.

让我们创建一个新的Mix项目.项目名称是`kv_umbrella`,这个新项目中既有已存在的`kv`应用,也有新的`kv_server`应用.它的目录结构会是这样:

```
+ kv_umbrella
  + apps
    + kv
    + kv_server
```

有趣的地方是Mix为这样的项目提供了许多便捷性,例如能够用一句命令来编译和测试`apps`中所有的应用.然而,即使它们在`apps`中是排列在一起的,它们之间仍然是解耦的,所以你可以随意独立地构建,测试和部署每个应用.

让我们开始吧!

#雨伞计划

让我们使用`mix new`开始新项目.这个新项目名为`kv_umbrella`,创建时我们还需要传递一个`--umbrella`选项.不要在已存在的`kv`项目中创建这个新项目!

```
$ mix new kv_umbrella --umbrella
* creating .gitignore
* creating README.md
* creating mix.exs
* creating apps
* creating config
* creating config/config.exs
```

从打印出的信息中,我们看到很少的文件被生成.生成的`mix.exs`文件也不同.让我们来看看(注释已删除):

```
defmodule KvUmbrella.Mixfile do
  use Mix.Project

  def project do
    [apps_path: "apps",
     build_embedded: Mix.env == :prod,
     start_permanent: Mix.env == :prod,
     deps: deps]
  end

  defp deps do
    []
  end
end
```

使得该项目于之前的项目不同的原因就是`apps_path`:项目定义中的`"apps"`条目.这意味着该项目会像一个雨伞一样运作.这样的项目没有源文件或测试,尽管他们可以由自己的依赖(不与孩子们共用).我们将在apps目录中创建新应用.

让我们进入apps目录,并开始构建`kv_server`.这一次,我们将传送`--sup`旗帜,它会让Mix保证为我们自动生成一个监督树,而不是像之前那样手动构建:

```
$ cd kv_umbrella/apps
$ mix new kv_server --module KVServer --sup
```

生成的文件与我们一开始为`kv`所生成的相似,只有一些不同.让我们打开`mix.exs`:

```
defmodule KVServer.Mixfile do
  use Mix.Project

  def project do
    [app: :kv_server,
     version: "0.1.0",
     build_path: "../../_build",
     config_path: "../../config/config.exs",
     deps_path: "../../deps",
     lockfile: "../../mix.lock",
     elixir: "~> 1.3",
     build_embedded: Mix.env == :prod,
     start_permanent: Mix.env == :prod,
     deps: deps]
  end

  def application do
    [applications: [:logger],
     mod: {KVServer, []}]
  end

  defp deps do
    []
  end
end
```

首先,因为我们是在`kv_umbrella/apps`中生成的这个项目,Mix自动检测到了雨伞结构并添加了四行代码到项目定义中:

```
build_path: "../../_build",
config_path: "../../config/config.exs",
deps_path: "../../deps",
lockfile: "../../mix.lock",
```

这些选项意味着所有依赖将被签出至`kv_umbrella/deps`,并且它们会分享相同的构建,配置和锁文件.这保证了整个雨伞结构中依赖只会被获取并编译一次,而不是每个应用都要.

第二个改变是`mix.exs`中的`application`函数:

```
def application do
  [applications: [:logger],
   mod: {KVServer, []}]
end
```

因为我们传送了`--sup`旗帜,Mix自动添加了`mod: {KVServer, []}`,将`KVServer`指定为了我们的应用回调模块.`KVServer`将会启动我们的应用监督树.

事实上,让我们打开`lib/kv_server.ex`:

```
defmodule KVServer do
  use Application

  def start(_type, _args) do
    import Supervisor.Spec, warn: false

    children = [
      # worker(KVServer.Worker, [arg1, arg2, arg3])
    ]

    opts = [strategy: :one_for_one, name: KVServer.Supervisor]
    Supervisor.start_link(children, opts)
  end
end
```

注意它定义了应用回调函数,`start/2`,并使用了`Supervisor`模块,而不是定义一个名为`KVServer.Supervisor`的主管,内联地定义一个主管很方便!你可以通过阅读Supervisor模块文档来获取更多关于这种主管的信息.

我们已经可以试试我们的第一个雨伞孩子.我们能在`apps/kv_server`目录中运行测试,但是并不有趣.让我们转到雨伞项目的根文件并运行`mix test`:

```
$ mix test
```

运行成功!

由于我们希望`kv_server`最后能使用我们在`kv`中定义的功能,所以我们需要将`kv`作为一个依赖添加到我们的应用中.

#伞内依赖

Mix提供了一个简单的机制来使一个伞孩子能够依赖于另一个.打开`apps/kv_server/mix.exs`并修改`deps/0`函数:

```
defp deps do
  [{:kv, in_umbrella: true}]
end
```

上述代码使得`kv`可以作为一个`:kv_server`中的依赖.我们可以导入`kv`中定义的而模块,但不会自动启动`:kv`应用.所以,我们也需要在`application/0`中列出`:kv`:

```
def application do
  [applications: [:logger, :kv],
   mod: {KVServer, []}]
end
```

现在Mix将会保证`:kv`应用在`:kv_server`启动之前启动.

最后,复制我们已经构建了的`kv`应用到我们的雨伞项目中的`apps`目录里.最终的目录结构是和我们之前提到的一样:

```
+ kv_umbrella
  + apps
    + kv
    + kv_server
```

现在我们只需要修改`apps/kv/mix.exs`来包含我们之前在`apps/kv_server/mix.exs`中看到的雨伞条目.打开`apps/kv/mix.exs`并添加到`project`函数:

```
build_path: "../../_build",
config_path: "../../config/config.exs",
deps_path: "../../deps",
lockfile: "../../mix.lock",
```

现在你可以在雨伞的根目录运行`mix test`来对两个项目进行测试了.欧耶!

记住,雨伞项目是一个帮助你组织和管理应用的便捷方法.`apps`目录中的应用仍然是互相解耦的.它们之间的依赖必须被明确列出.它们可以被共同开发,但是如果需要的话,可以独立地被编译,测试和部署.

#总结

本章我们学习了更多关于Mix依赖和雨伞项目的内容.我们决定构建一个雨伞项目,是因为我们认为`kv`和`kv_server`是只在本项目中有用的内部依赖.

未来,我们将写一些应用,你会注意到它们可以被提取到一个简洁的单元中,然后被不同的项目使用.这时,就应该使用Git或Hex依赖了.

这里有一些问题,当处理依赖时你可以问自己.开始:这个应用在此项目之外还有意义吗?

  \- 如果没有,使用带伞孩子的雨伞项目
  \- 如果有,该项目是否可以分享到你的公司/组织之外?
    \- 如果不行,使用私有git仓库.
    \- 如果可以,将你的代码push到git仓库并经常使用Hex发布.

我们的雨伞项目已经构建并运行了,现在让我们开始编写服务器.
