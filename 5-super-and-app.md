5-监督者和应用程序
======================

- [第一个监督者]()  
- [理解应用程序]()  
  - [启动应用程序]()  
  - [应用程序的回调函数]()  
  - [工程还是应用程序？]()  
- [简单的一对一监督者]()  
- [监督树]()  

到目前为止，我们的程序需要一个事件管理器和一个注册表进程。它还会有不是成百，就是上千的bucket进程。
你是不是觉得这个还不错？没有东西是完美的，也许马上就要出现bug了。

当有东西挂了，我们的第一反应是：“快拯救这些错误”。但是，像在《入门》中学到的那样，不同于其它多数语言，Elixir不太做“防御性编程”。
相反，我们说“要挂快点挂”，或是“就让它挂”。
如果有bug要让我们的注册表进程挂掉，啥也不怕，因为我们即将用监督者启动一个新的注册表进程。

本章，我们就要学习监督者，还会讲到应用程序。一个不够，我们要创建两个监督者，用它们监督我们的进程。

## 5.1-第一个监督者
创建一个监督者跟创建GenServer比没多少不同。
定义一个名为```KV.Supervisor```的模块，这个模块使用[Supervisor](http://elixir-lang.org/docs/stable/elixir/Supervisor.html)行为抽象。此文件```lib/kv/supervisor.ex```的内容：
```elixir
defmodule KV.Supervisor do
  use Supervisor

  def start_link do
    Supervisor.start_link(__MODULE__, :ok)
  end

  @manager_name KV.EventManager
  @registry_name KV.Registry

  def init(:ok) do
    children = [
      worker(GenEvent, [[name: @manager_name]]),
      worker(KV.Registry, [@manager_name, [name: @registry_name]])
    ]

    supervise(children, strategy: :one_for_one)
  end
end
```

我们的监督者有两个孩子：事件管理器和注册表进程。通常会给监督者旗下的进程起个名字，好让别的进程用名字而不是pid访问它们。
这很有用，因为一个被监督的进程可能会挂，一挂再重启，pid就变了。  
用```@manager_name```和```@registry_name```这两个模块属性给我们监督者的俩孩子声明名字，然后在“工人（worker）”的定义中引用这两个属性值。尽管不是必须用模块属性来定义名字，但很实用，因为这样读起来很醒目。

举个例子，```KV.Registry```工人接受两个参数，第一个是事件管理器的名字，第二个是一个选项键值列表。
在这里，我们设置一个名字选项```[name: KV.Registry]```（使用之前定义的模块属性：```@registry_name```），确保我们可以在整个应用程序中通过名字```KV.Registry```访问注册表进程。用定义它们的模块名称给监督者孩子起名的做法十分普遍，因为在调试一个正则运行的系统时很有用。

监督者中孩子们声明的顺序也是有区别的。因为注册表依赖于事件管理器，我们必须先启动前者。这也是在孩子列表中，```GenEvent```工人的位置靠前的原因。

最后，我们调用了```supervisor/2```，给它传递了一个孩子列表，以及策略：```:one_for_one```。

监督的策略指明了当一个孩子进程挂了会发生什么。```:one_for_one```意思是如果一个孩子进程挂了，只有一个它的“复制品”会启动来替代它。
这个策略现在是说得通的。如果事件管理器挂了，没理由连注册表进程也重启。反之亦然。
但是如果监督者旗下的孩子越来越多时，这个策略就需要改变了。```Supervisor```行为抽象支持许多不同的策略，我们在本章中将会讨论其中三种。

如果我们在工程中启动命令行对话```iex -S mix```，我们可以手动启动监督者：
```elixir
iex> KV.Supervisor.start_link
{:ok, #PID<0.66.0>}
iex> KV.Registry.create(KV.Registry, "shopping")
:ok
iex> KV.Registry.lookup(KV.Registry, "shopping")
{:ok, #PID<0.70.0>}
```

当我们启动监督树，事件管理器和注册表进程都自动被启动，允许我们创建bucket。不再需要手动启动它们。

尽管在实战中，我们很少手动启动应用程序的监督者。相反，它的启动是应用程序回调的一部分。

## 5.2-理解应用程序
我们已经在这个应用程序上花了很多时间。每次修改了一个文件，执行```mix compile```，我们都能看到```Generated kv.app```消息打印出来。

我们可以在```_build/dev/lib/kv/ebin/kv.app```找到```.app```文件。我们来看一下它的内容：
```elixir
{application,kv,
             [{registered,[]},
              {description,"kv"},
              {applications,[kernel,stdlib,elixir,logger]},
              {vsn,"0.0.1"},
              {modules,['Elixir.KV','Elixir.KV.Bucket',
                        'Elixir.KV.Registry','Elixir.KV.Supervisor']}]}.
```
该文件包含Erlang的语句（使用Erlang的语法写的）。即使我们不熟悉Erlang，也能很容易猜到这个文件保存的是我们应用程序的定义。
它包括应用程序的版本，定义的所有模块，还有它依赖的应用程序列表，如Erlang的Kernel，elixir本身，logger（我们在```mix.exs```里添加的）。

要是每次我们添加一个新的模块就要手动修改这个文件，是很讨厌的。这也是为啥把它交给mix来自动维护的原因。

我们还可以通过修改```mix.exs```工程文件中的函数```application/0```的返回值，来配置产生的```.app```文件。
我们会在接下来几章讲到这个。

### 5.2.1-启动应用程序
定义了```.app```文件（里面是应用程序的定义），我们就可以将应用程序视作一个整体形式来启动和停止。
到目前为止我们还没有考虑过这个问题，这是因为：  

1. Mix为我们自动启动了应用程序
2. 即使Mix不自动启动我们的程序，要启动该程序时也不需要做啥特别的事儿

总之，让我们看看Mix如何为我们启动应用程序，先启动工程下的命令行，然后试着：
```elixir
iex> Application.start(:kv)
{:error, {:already_started, :kv}}
```

擦，已经启动了？

我们可以给mix一个选项，让它不要启动我们的应用程序。命令：```iex -S mix run --no-start```：
```elixir
iex> Application.start(:kv)
{:error, {:not_started, :logger}}
```

这次我们得到的错误是由于```:kv```所依赖的应用程序（这里是```:logger```）没有启动。
Mix一般会根据工程中的```mix.exs```启动整个应用程序结构；对其依赖的每个应用程序来说也是这样（如果它们还依赖于其它应用程序）。
但是这次我们用了```--no-start```标志，因此我们需要手动按顺序启动所有应用程序，或者像这样调用```Application.ensure_all_started```:
```elixir
iex> Application.ensure_all_started(:kv)
{:ok, [:logger, :kv]}
iex> Application.stop(:kv)
18:12:10.698 [info] Application kv exited :stopped
:ok
```
没什么激动人心的，但是这演示了如何控制我们的应用程序。

>当你运行```iex -S mix```，它相当于```iex -S mix run```。因此无论何时你启动iex会话，传递参数给```mix run```，实际上是传递给```run```命令。你可以在命令行中执行```mix help run```获取关于```run```的更多信息。

### 5.2.2-应用程序的回调函数
因为我们几乎都在讲应用程序如何启动和停止，你能猜到肯定有某个方法能在启动的当儿做点有意义的事情。没错，有的！

我们可以定义应用程序的回调函数。在应用程序启动时，该函数将被调用。
这个函数必须返回```{:ok, pid}```，其中```pid```是其内部监督者进程的标识符。

我们分两步来定义这个回调函数。首相，打开```mix.exs```文件，修改```def application```部分为：
```elixir
def application do
  [applications: [],
   mod: {KV, []}]
end
```

选项```:mod```指出了“应用程序回调函数的模块”，后面跟着个在应用程序启动是会被传递过来的参数。
这个回调函数的模块可以是任意的，只要它实现了[Application](http://elixir-lang.org/docs/stable/elixir/Application.html)行为。

现在，我们让```KV```来做这个回调函数的模块。在文件```lib/kv.ex```中，做一下修改：
```elixir
defmodule KV do
  use Application

  def start(_type, _args) do
    KV.Supervisor.start_link
  end
end
```

当我们```use Application```，我们仅仅需要定义一个```start/2```函数。
而如果我们想在应用程序停止时定义一个自定义的行为，我们__也__可以定义一个```stop/1```函数。
在这里，我们不这么做，就用其默认的、自动定义在```use Application```中的行为。

现在我们再次用```iex -S mix```启动我们的工程对话。我们将开到一个名为```KV.Registry```的进程已经在运行：
```elixir
iex> KV.Registry.create(KV.Registry, "shopping")
:ok
iex> KV.Registry.lookup(KV.Registry, "shopping")
{:ok, #PID<0.88.0>}
```
好牛逼！

### 5.2.3-工程还是应用程序？
Mix是区分工程（projects）和应用程序（applications）的。
基于目前的```mix.exs```，我们可以说，我们有一个Mix __工程__，该工程定义了```:kv```应用程序。
在后面章节我们会看到，有些工程一个应用程序也没定义。

当我们讲“工程”时，你应该想到Mix。Mix是管理工程的工具。
它知道如何去编译、测试你的工程，等等。它还知道如何编译和启动你的工程的相关应用程序。

当我们讲“应用程序”时，我们讨论的是OTP。应用程序是一个实体，它作为一个整体启动或者停止。
你可以在[应用程序模块文档](http://elixir-lang.org/docs/stable/elixir/Application.html)阅读更多关于应用程序的知识。

## 5.3 简单的一对一监督者
我们已经成功定义了我们的监督者，它作为我们应用程序生命周期的一部分自动启动（和停止）。

回一下，我们的```KV.Registry```在```handle_cast/2```回调中，链接并且监视bucket进程：
```elixir
{:ok, pid} = KV.Bucket.start_link()
ref = Process.monitor(pid)
```
链接是双向的，意味着一个bucket进程挂了会导致注册表进程挂掉。尽管现在我们有了监督者，它能保证一旦注册表进程挂了还可以重启，
但是我们保存在bucket相应进程的数据还是会丢失。

换句话说，我们希望即使bucket进程挂了，注册表进程也能够保持运行。写个测试：
```elixir
test "removes bucket on crash", %{registry: registry} do
  KV.Registry.create(registry, "shopping")
  {:ok, bucket} = KV.Registry.lookup(registry, "shopping")

  # Kill the bucket and wait for the notification
  Process.exit(bucket, :shutdown)
  assert_receive {:exit, "shopping", ^bucket}
  assert KV.Registry.lookup(registry, "shopping") == :error
end
```

这个测试很像之前的“退出时移除bucket”，只是我们的做法更加有点问题。
我们没有使用```Agent.stop/1```，而是发送了一个退出信号来关闭bucket进程。因为bucket是链接在注册表进程的，而注册表进程是连接着测试进程，让bucket挂回导致连测试进程都挂掉：
```
1) test removes bucket on crash (KV.RegistryTest)
   test/kv/registry_test.exs:52
   ** (EXIT from #PID<0.94.0>) shutdown
```

一个可行的解决方法是提供```KV.Bucket.start/0```，让它执行```Agent.start/1```。
在注册表进程中使用这个方法启动bucket，从而去掉了它们之间的链接。
但是这不是个好办法，因为这样bucket进程就链接不到任何进程。这意味着所有bucket进程即使在不可访问的状态下也一直活着。

我们将定义一个新的监督者来解决这个问题。这个新监督者来派生和监督所有的bucket。
有一个简单的一对一监督策略，叫做```:simple_one_for_one```，对于此情况是非常适用的：
他允许指定一个工人模板，而后监督基于那个模板的多个孩子。

让我们定义```KV.Bucket.Supervisor```：
```elixir
defmodule KV.Bucket.Supervisor do
  use Supervisor

  def start_link(opts \\ []) do
    Supervisor.start_link(__MODULE__, :ok, opts)
  end

  def start_bucket(supervisor) do
    Supervisor.start_child(supervisor, [])
  end

  def init(:ok) do
    children = [
      worker(KV.Bucket, [], restart: :temporary)
    ]

    supervise(children, strategy: :simple_one_for_one)
  end
end
```

比起之前的，这个监督者有两点改变。

第一，我们定义了```start_bucket/1```函数，接受一个监督者id，并且启动一个bucket进程作为该监督者的孩子。```start_bucket/1```代替了之前在注册表进程中直接调用的```KV.Bucket.start_link```。

第二，在```init/1```回调中，我们将工人标记为```:temporary```。意思是如果bucket挂了，它不会重启！
这是因为我们只是想用监督者将bucket组织起来。创建bucket还得通过注册表进程。

运行```iex -S mix```，试一试我们的新监督者：
```elixir
iex> {:ok, sup} = KV.Bucket.Supervisor.start_link
{:ok, #PID<0.70.0>}
iex> {:ok, bucket} = KV.Bucket.Supervisor.start_bucket(sup)
{:ok, #PID<0.72.0>}
iex> KV.Bucket.put(bucket, "eggs", 3)
:ok
iex> KV.Bucket.get(bucket, "eggs")
3
```

现在来对注册表进程稍作修改，以配合bucket的监督者。
我们会沿用和处理事件管理器时相同的策略，会显式地传递bucket的监督者的pid给```KV.Registery.start_link/3```。
让我们先从修改```test/kv/registry_test.exs```中的Setup回调开始：
```elixir
setup do
  {:ok, sup} = KV.Bucket.Supervisor.start_link
  {:ok, manager} = GenEvent.start_link
  {:ok, registry} = KV.Registry.start_link(manager, sup)

  GenEvent.add_mon_handler(manager, Forwarder, self())
  {:ok, registry: registry}
end
```
接下来开始修改```KV.Registry```中的函数，将新监督者投入使用：
```elixir
## Client API

@doc """
Starts the registry.
"""
def start_link(event_manager, buckets, opts \\ []) do
  # 1. Pass the buckets supevisor as argument
  GenServer.start_link(__MODULE__, {event_manager, buckets}, opts)
end

## Server callbacks

def init({events, buckets}) do
  names = HashDict.new
  refs  = HashDict.new
  # 2. Store the buckets supevisor in the state
  {:ok, %{names: names, refs: refs, events: events, buckets: buckets}}
end

def handle_cast({:create, name}, state) do
  if HashDict.get(state.names, name) do
    {:noreply, state}
  else
    # 3. Use the buckets supervisor instead of starting buckets directly
    {:ok, pid} = KV.Bucket.Supervisor.start_bucket(state.buckets)
    ref = Process.monitor(pid)
    refs = HashDict.put(state.refs, ref, name)
    names = HashDict.put(state.names, name, pid)
    GenEvent.sync_notify(state.events, {:create, name, pid})
    {:noreply, %{state | names: names, refs: refs}}
  end
end
```
改这些基本上就能让测试通过了。要完成任务，只需再修改下这个监督者，让原来bucket的那个监督者也成为它的孩子。

## 5.4-监督树

为了使用bucket的监督者，我们要把它作为一个孩子加到```KV.Supervisor```中去。
注意，我们已经开始用一个监督者去监督另一个监督者了。正式的称呼是“监督树”。

打开```lib/kv/supervisor.ex```，添加一个新的模块属性存储bucket监督者的名字，并且修改```init/1```：
```elixir
@manager_name KV.EventManager
@registry_name KV.Registry
@bucket_sup_name KV.Bucket.Supervisor

def init(:ok) do
  children = [
    worker(GenEvent, [[name: @manager_name]]),
    supervisor(KV.Bucket.Supervisor, [[name: @bucket_sup_name]]),
    worker(KV.Registry, [@manager_name, @bucket_sup_name, [name: @registry_name]])
  ]

  supervise(children, strategy: :one_for_one)
end
```

这一次，我们添加了一个监督者作为孩子，并且给了它一个名字```KV.Bucket.Supervisor```（和它的模块名相同）。
我们还更新了```KV.Registry```这个工人，使它接受bucket监督者的名字作为参数。

记住，声明各个孩子的顺序是很重要的。因为注册表进程依赖于bucket监督者，所以bucket监督者需要在孩子列表中排得靠前一些。

因为我们已为监督者添加了多个孩子，现在就需要考虑使用```:one_for_one```这个策略还是否正确。
一个显现的问题就是注册表进程和bucket监督者之间的关系。如果注册表进程挂了，bucket监督者也必须挂。
因为一旦注册表进程挂了，所有关联bucket名字和其进程的信息也就丢失了。此时若bucket的监督者还活着，它掌管的众多bucket根本访问不到，变成垃圾。

我们可以考虑使用其他的策略，如```:one_for_all```。这个策略在任何时候，只要有一个孩子挂，它就会停止并且重启所有孩子进程。
这也许不是最理想的，因为注册表进程挂了不应该影响到事件管理器。
事实上，这么做也是有害的，因为要是事件管理器挂了，所有注册的事件处理者也都跟着会被移除。

一个可行的方案是再新建一个监督者，让它来监督注册表进程和bucket监督者，使用```:one_for_all```策略。
让我们的根监督者监督这个新建的，以及事件管理器这两个孩子，使用```:one_for_one```策略。
这个监督树大概是一下这个样子：
```
* root supervisor [one_for_one]
  * event manager
  * supervisor [one_for_all]
    * buckets supervisor [simple_one_for_one]
      * buckets
    * registry
```

你可以试着构建这个监督树，但我们不会再对它进行改进了。
因为下一章我们会修改注册表进程，让它可以持久化（persist）它的注册信息，使得我们用的```:one_for_one```策略成为最合适的策略，省得新增监督树结构或者改变策略了。

记住，还有几个策略可以传递给```worker/2```，```supervisor/2```和```supervise/2```函数，所以可别忘记阅读[监督者模块的文档](http://elixir-lang.org/docs/stable/elixir/Supervisor.html).























































