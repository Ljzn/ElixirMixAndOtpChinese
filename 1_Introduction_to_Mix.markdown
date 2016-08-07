#Mix入门

  1. 我们的第一个项目
  2. 编辑项目
  3. 执行测试
  4. 环境
  5. 探索

在本教程中,我们将学习如何构建一个完整的Elixir应用,包括监督树,配置,测试等等.

这个应用的功能是分布式键值仓库.我们将把键值对安排到桶中,并将桶分布到多个节点.我们也会构建一个简单的客户端,让我们能够与其中任何一个节点连接并发送如下请求:

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

为了构建我们的键值应用,我们会用到三个主要工具:

  - **OTP**(_开放通信平台_)是Erlang装载的一系列库.Erlang开发者使用OTP来构建健壮的,容错的应用.本章我们将探索Elixir融合了多少来自OTP的内容,包括监督树,事件管理等等;

  - **Mix** 是Elixir装载的一个构建工具,提供了创建,编译,测试应用,管理依赖等等;

  - **ExUnit** 是Elixir装载的一个基本单元测试框架;

本章,我们将使用Mix创建第一个项目,并探索OTP,Mix和ExUnit的不同特性.

> 注意:本教程要求Elixir版本v1.2.0或以上.你可以用`elixir -v`来检查版本,如有需要可按入门教程的第一章中的步骤安装最新的版本.

> 如对本教程有任何疑问或改进意见,请通过我们的邮件列表或问题跟踪来告诉我们.你的意见对于帮助我们确保这份教程可用且最新十分重要!

#我们的第一个项目

当你安装Elixir时,除了得到了`elixir`,`elixirc`和`iex`这些可执行文件外,还得到了一个可执行的Elixir脚本`mix`.

让我们通过从命令行中调用`mix new`来创建我们的第一个项目.我们会将项目名当做参数传递(本例中是`kv`),并告诉Mix我们的主模块是全大写的`KV`,而不是默认的`Kv`:

```
$ mix new kv --module KV
```

Mix会创建一个叫`kv`的目录,其中有如下文件:

```
* creating README.md
* creating .gitignore
* creating mix.exs
* creating config
* creating config/config.exs
* creating lib
* creating lib/kv.ex
* creating test
* creating test/test_helper.exs
* creating test/kv_test.exs
```

让我们简单查看一下这些文件.

> 注意: Mix是一个Elixir可执行文件.这意味着想要运行`mix`,你的PATH中需要有Elixir的可执行文件.如果没有,你可以将脚本当作参数传递给`elixir`来运行它:
```
$ bin/elixir bin/mix new kv --module KV
```
注意你也可以在PATH中用Elixir加上-S选项来执行任何脚本:
```
$ bin/elixir -S mix new kv --module KV
```
当使用-S时,`elixir`会找到脚本并执行它,无论是否存在与你的PATH中.

#编辑项目

在我们的新项目目录(`kv`)中生成了一个叫`mix.exs`的文件,它的主要作用是配置我们的项目.让我们来看看(注释已删除):

```
defmodule KV.Mixfile do
  use Mix.Project

  def project do
    [app: :kv,
     version: "0.1.0",
     elixir: "~> 1.3",
     build_embedded: Mix.env == :prod,
     start_permanent: Mix.env == :prod,
     deps: deps()]
  end

  def application do
    [applications: [:logger]]
  end

  defp deps do
    []
  end
end
```

我们的`mix.exs`定义了两个公共函数:`project`,它会返回诸如项目名称和版本等项目配置,以及`application`,它用于生成应用文件.

这里还有一个叫做`deps`的私有函数,它被`project`函数调用,定义了我们的项目的依赖.将`deps`作为一个独立的函数不是必需的,但这样做可以有助于保持项目配置的整洁.

Mix也生成了一个文件`lib/kv.ex`,其中有一个简单的模块定义:

```
defmodule KV do
end
```

这个结构体足够编译我们的项目:

```
$ cd kv
$ mix compile
```

将输出:Compiling 1 file (.ex) Generated kv app

`lib/kv.ex`已被编译,生成了一个名为`kv.app`的应用,并且所有的协议都如入门教程中描述的那样被巩固了.所有的编译成品都如`mix.exs`中定义的那样被存放在了`_build`目录中.

编译完成后,你就可以在项目内开启一个`iex`会话,通过运行:

```
$ iex -S mix
```

#运行测试

Mix也为运行我们的项目测试而生成了合适的结构.通常,Mix项目以方便起见会在`lib`目录中的`test`目录下为每个文件生成一个`<filename>_test.exs`文件.所以,我们能够找到一个`test/kv_test.exs`文件对应着我们的`lib/kv.ex`文件.这一点上做的不多:

```
defmodule KVTest do
  use ExUnit.Case
  doctest KV

  test "the truth" do
    assert 1 + 1 == 2
  end
end
```

这有几件事要特别注意的:

  1. 测试文件是一个Elixir脚本文件(`.exs`).便捷之处在于我们不需要在运行测试之前编译它们;

  2. 我们定义了一个名为`KVTest`的模块,使用`ExUnit.Case`来注入测试API,并使用`test/2`宏定义了一个简单的测试;

Mix也生成了一个叫`test/test_helper.exs`的文件,它的作用是启动测试框架:

```
ExUnit.start()
```

这个文件会在每次我们运行测试之前被Mix自动调用.我们可以使用`mix test`运行测试:

```
Compiled lib/kv.ex
Generated kv app
[...]
.

Finished in 0.04 seconds (0.04s on load, 0.00s on tests)
1 test, 0 failures

Randomized with seed 540224
```

注意,通过运行`mix test`,Mix再一次编译了源文件并生成应用.这是由于Mix支持多重环境,我们将在下一部分探索它.

而且,你可以看到ExUnit为每个成功的测试打印了一个点,并且自动进行了乱序测试.试试让测试失败会发生什么.

将`test/kv_test.exs`中的断言改成:

```
assert 1 + 1 == 3
```

再次运行`mix test`(注意这次没有进行编译):

```
1) test the truth (KVTest)
   test/kv_test.exs:5
   Assertion with == failed
   code: 1 + 1 == 3
   lhs:  2
   rhs:  3
   stacktrace:
     test/kv_test.exs:6

Finished in 0.05 seconds (0.05s on load, 0.00s on tests)
1 test, 1 failure
```

对于每个失败,ExUnit都打印了一个详细的报告,包括测试名称与测试案例,失败的代码和`==`符号左手边(lhs)与右手边(rhs)的值.

在失败的第二行,测试名之后,是测试定义的位置.如果你复制了第二行(包括文件名与行号)并将其添加到`mix test`之后,Mix将会只载入和运行这一测试:

```
$ mix test test/kv_test.exs:5
```

这个短语在我们构建项目时非常有用,它让我们能够快速地重复某一特定测试.

最后,堆栈跟踪指向了失败本身,提供了与测试相关的信息,以及源文件中失败生成的位置.

#环境

Mix支持"环境"的概念.它们允许开发者为特定的情景自定义编译和其它选项.Mix默认接受三种环境:

  - `:dev`--Mix任务(例如`compile`)按默认设置运行
  - `:test`--由`mix test`使用
  - `:prod`--在项目产品运行时用到的

环境设置只对当前项目生效.正如我们将要看到的,任何你添加到项目中的依赖会默认运行在`:prod`环境.

你可以通过访问`mix.exs`文件中的`Mix.env`函数来自定义环境,它会以原子形式返回当前环境.这就是我们在`:bulid_embedded`和`:start_permanent`选项中所使用的:

```
def project do
  [...,
   build_embedded: Mix.env == :prod,
   start_permanent: Mix.env == :prod,
   ...]
end
```

当你编译源代码时,Elixir将编译成果放到额`_build`目录中.然而,为了避免不必要的复制,Elixir会创建一个从`_build`到实际源代码文件的文件系统链接.当`:build_embedded`为真时,会停止这个行为,意在使所有运行应用所需的东西都放在`_build`中.

相似地,当`:start_permanent`选项为真时,你的应用将运行在永久模式,这意味着如果应用的监督树关闭了,Erlang虚拟机就会崩溃.注意我们不希望这个行为出现在dev和test环境中,因为保持Erlang虚拟机运行对于解决问题十分有用.

Mix默认处于`:dev`环境,只有`test`任务会默认在`:test`环境.可以通过`MIX_ENV`环境变量来修改环境:

```
$ MIX_ENV=prod mix compile
```

在Windows中:

```
> set "MIX_ENV=prod" && mix compile
```

#探索

Mix还有很多内容,我们将在构建项目的过程中继续探索它.在Mix文档中你能得到一个概述.

记住你总能通过调用帮助任务来列出所有可用任务:

```
$ mix help
```

你可以通过调用`mix help TASK`来获得有关特定任务的信息.

让我们开始写代码吧!
