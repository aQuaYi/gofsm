# gofsm

[![License](https://img.shields.io/:license-apache-blue.svg)](LECENSE)
[![GoDoc](https://godoc.org/github.com/aQuaYi/gofsm?status.png)](http://godoc.org/github.com/aQuaYi/gofsm)
![travis](https://travis-ci.org/aQuaYi/gofsm.svg?branch=master)
[![Coverage](http://gocover.io/_badge/github.com/aQuaYi/gofsm)](http://gocover.io/github.com/aQuaYi/gofsm)
[![Go Report Card](https://goreportcard.com/badge/github.com/aQuaYi/gofsm)](https://goreportcard.com/report/github.com/aQuaYi/gofsm)

本项目基于 [smallnest 的 gofsm](https://github.com/smallnest/gofsm) 整理修改而来，是一个简单、小巧而又特色的有限状态机（FSM）。
和[其他 Go 语言 FSM 项目](https://github.com/search?l=Go&o=desc&q=fsm&s=stars&type=Repositories)相比较，本 FSM 有以下特点:

1. gofsm 本身是 `stateless`，只负责转换状态的逻辑，与具体对象的状态维护代码实现了正交。
1. 提供了 [Moore](https://zh.wikipedia.org/wiki/%E6%91%A9%E5%B0%94%E5%9E%8B%E6%9C%89%E9%99%90%E7%8A%B6%E6%80%81%E6%9C%BA) 和 [Mealy](https://zh.wikipedia.org/wiki/%E7%B1%B3%E5%88%A9%E5%9E%8B%E6%9C%89%E9%99%90%E7%8A%B6%E6%80%81%E6%9C%BA) 两种状态机的统一接口和 UML 状态机风格的 Action 处理方式。
1. 可以生成具体对象的状态转换图，易于检查代码。

![gofsm生成的闸门状态图](state.png)

## 有限状态机

`有限状态机`（finite-state machine）是描述事物运行变化过程的有力工具。它把事物在整个生命周期过程，分割成有限个状态。任意时刻这个机器只有唯一的一个状态，称为`当前状态`。触发转换条件后，它的状态会发生改变，称为`转换`（transition）。一个 FSM 就是由初始状态、所有的状态和每个转换的触发条件所定义。有时候，当这个转换发生的时候，我们还可以要执行一些事情，称为`动作`（Action）。

生活中，我们经常会遇到 FSM，比如路口的红绿灯，总是在红、黄、绿的状态之间转变，比如电梯的状态，包括开、关、上、下等几个状态。

![旋转门](turnstile.jpeg)

以`旋转门`(turnstile)为例。投币前，闸门紧缩，推不动它的转臂，投币后，只能一个人通过，之后闸门再次锁定。

![旋转门的状态图](fsm-of-turnstile.png)

它有两个状态：`Locked`、`Unlocked`。有两个输入（input）会影响它的状态，`投币`(coin)和`推动转臂`（push）。

1. 在 Locked 状态， Push 不会改变旋转门的状态。
2. 在 Locked 状态， Coin 会让旋转门变成 Unlocked 状态。
3. 在 Unlocked 状态， Coin 不起作用，旋转门依然是 Unlocked 状态。
4. 在 Unlocked 状态， Push 后旋转门会由转变成 Locked 状态。

这是一个简单的闸门的状态转换，却是一个很好的理解状态的典型例子。

以表格来表示：

<!-- markdownlint-disable MD033 -->

<table><tbody><tr><th>Current State</th><th>Input</th><th>Next State</th><th>Output</th></tr><tr><th rowspan="2">Locked</th><td>coin</td><td>Unlocked</td><td>Unlock turnstile so customer can push through</td></tr><tr><td>push</td><td>Locked</td><td>None</td></tr><tr><th rowspan="2">Unlocked</th><td>coin</td><td>Unlocked</td><td>None</td></tr><tr><td>push</td><td>Locked</td><td>When customer has pushed through, lock turnstile</td></tr></tbody></table>

UML 也有状态图的改变，它扩展了 FSM 的概念，提供了层次化的嵌套状态（Hierarchically nested states）和正交区域（orthogonal regions），当然这和本文没有太多的关系，有兴趣的读者可以找 UML 的资料看看。但是它提供了一个很好的概念，也就是动作（Action）。就像 Mealy 状态机所需要的一样，动作依赖系统的状态和触发事件，而它的Entry Action 和 Exit Action，却又像 Moore 状态机一样，不依赖输入，只依赖状态。所以UML的动作有三种，一种是事件被处理的时候，状态机会执行特定的动作，比如改变变量、执行I/O、调用方法、触发另一个事件等。而离开一个状态，可以执行Exit action，进入一个状态，则执行Entry action。记住，收到一个事件，对象的状态不会改变，比如上边闸门的例子，在 Locked 状态下 push 多少次状态都没改变，在这种情况下，不会执行 Exit 和 Entry action。

gofsm提供了这种扩展的模型，当然如果你不想使用这种扩展，你也可以不去实现 Entry 和 Exit。

可以提到了两种状态机，这两种状态机是这样来区分的：

* **Moore** 状态机只使用 entry action，输出只依赖状态，不依赖输入。
* **Mealy** 状态机只使用 input action，输出依赖输入 input 和状态 state。使用这种状态机通常可以减少状态的数量。

gofsm提供了一个通用的接口，你可以根据需要确定使用哪个状态机。从软件开发的实践上来看，有时候你并不一定要关注状态机的区分，而是清晰的抽象、设计你所关注的对象的状态、触发条件以及要执行的动作。

## gofsm 代码

gofsm参考了 [elimisteve/fsm](https://github.com/elimisteve/fsm) 的实现，实现了一种单一状态机处理多个对象的方法，并且提供了输出状态图的功能。

它除了定义对象的状态外，还定义了触发事件以及处理的Action，这些都是通过字符串来表示的，在使用的时候很容易的和你的对象、方法对应起来。

使用gofsm也很简单，当然第一步将库拉到本地：

```sh
go get -u github.com/aQuaYi/gofsm
```

我们以上面的闸门为例，看看gofsm是如何使用的。

注意下面的单个状态机可以处理并行地的处理多个闸门的状态改变，虽然例子中只生成了一个闸门对象。

首先定义一个闸门对象,它包含一个State，表示它当前的状态：

```go
type Turnstile struct {
  ID         uint64
  EventCount uint64 //事件统计
  CoinCount  uint64 //投币事件统计
  PassCount  uint64 //顾客通过事件统计
  State      string //当前状态
  States     []string //历史经过的状态
}
```

状态机的初始化简单直接：

```go
func initFSM() *StateMachine {
  delegate := &DefaultDelegate{P: &TurnstileEventProcessor{}}

  transitions := []Transition{
    Transition{From: "Locked", Event: "Coin", To: "Unlocked", Action: "check"},
    Transition{From: "Locked", Event: "Push", To: "Locked", Action: "invalid-push"},
    Transition{From: "Unlocked", Event: "Push", To: "Locked", Action: "pass"},
    Transition{From: "Unlocked", Event: "Coin", To: "Unlocked", Action: "repeat-check"},
  }

  return NewStateMachine(delegate, transitions...)
}
```

你定义好转换对应关系 `transitions` ,一个 `Transition` 代表一个转换，从某个状态到另外一个状态，触发的事件名，要执行的Action。
因为Action是字符串，所以你需要实现 `delegate` 将Action和对应的要处理的方法对应起来。

注意 from 和 to 的状态可以一样，在这种情况下，状态没有发生改变，只是需要处理Action就可以了。

如果Action为空，也就是不需要处理事件，只是发生状态的改变而已。

处理Action的类型如下：

```go
type TurnstileEventProcessor struct{}

func (p *TurnstileEventProcessor) OnExit(fromState string, args []interface{}) {
  // ...
}

func (p *TurnstileEventProcessor) Action(action string, fromState string, toState string, args []interface{}) {
  // ...
}

func (p *TurnstileEventProcessor) OnEnter(toState string, args []interface{}) {
  // ...
}
```

然后我们就可以触发一些事件看看闸门的状态机是否正常工作：

```go
  ts := &Turnstile{
    ID:     1,
    State:  "Locked",
    States: []string{"Locked"},
  }
  fsm := initFSM()

  //推门
  //没刷卡/投币不可进入
  err := fsm.Trigger(ts.State, "Push", ts)
  if err != nil {
    t.Errorf("trigger err: %v", err)
  }

  //推门
  //没刷卡/投币不可进入
  err = fsm.Trigger(ts.State, "Push", ts)
  if err != nil {
    t.Errorf("trigger err: %v", err)
  }

  //刷卡或者投币
  //不容易啊，终于解锁了
  err = fsm.Trigger(ts.State, "Coin", ts)
  if err != nil {
    t.Errorf("trigger err: %v", err)
  }

  //刷卡或者投币
  //这时才解锁
  err = fsm.Trigger(ts.State, "Coin", ts)
  if err != nil {
    t.Errorf("trigger err: %v", err)
  }

  //推门
  //这时才能进入，进入后闸门被锁
  err = fsm.Trigger(ts.State, "Push", ts)
  if err != nil {
    t.Errorf("trigger err: %v", err)
  }

  //推门
  //无法进入，闸门已锁
  err = fsm.Trigger(ts.State, "Push", ts)
  if err != nil {
    t.Errorf("trigger err: %v", err)
  }

  lastState := Turnstile{
    ID:         1,
    EventCount: 6,
    CoinCount:  2,
    PassCount:  1,
    State:      "Locked",
    States:     []string{"Locked", "Unlocked", "Locked"},
  }

  if !compareTurnstile(&lastState, ts) {
    t.Errorf("Expected last state: %+v, but got %+v", lastState, ts)
  } else {
    t.Logf("最终的状态: %+v", ts)
  }
```

如果想将状态图输出图片，可以调用下面的方法，它实际是调用 graphviz 生成的，所以请确保你的机器上是否安装了这个软件，你可以执行`dot -h`检查一下：

```go
  fsm.Export("state.png")
```

生成的图片就是文首的闸门的状态机的图片。

如果你想定制graphviz的参数，你可以调用另外一个方法：

```go
func (m *StateMachine) ExportWithDetails(outfile string, format string, layout string, scale string, more string) error
```

## 其它 Go 语言实现的 FSM

如果你发现gofsm的功能需要改进，或者有一些想法、或者发现了bug，请不用迟疑，在[issue](https://github.com/aQuaYi/gofsm/issues)中提交你的意见和建议，我会及时的进行反馈。

如果你觉得本项目有用，或者将来可能会使用，请star这个项目 [aQuaYi/gofsm](https://github.com/aQuaYi/gofsm)。

如果你想比较其它的 Go 语言实现的 fsm，可以参考下面的列表：

* [elimisteve/fsm](https://github.com/elimisteve/fsm)
* [looplab/fsm](https://github.com/looplab/fsm)
* [vaughan0/go-fsm](https://github.com/vaughan0/go-fsm)
* [WatchBeam/fsm](https://github.com/WatchBeam/fsm)
* [DiscoViking/fsm](https://github.com/DiscoViking/fsm)
* [autocube/hsm](https://github.com/autocube/hsm)
* [theckman/go-fsm](https://github.com/theckman/go-fsm)
* [Zumata/fsm](https://github.com/Zumata/fsm)
* [syed/go-fsm](https://github.com/syed/go-fsm)
* [go-rut/fsm](https://github.com/go-rut/fsm)
* [yandd/fsm](https://github.com/yandd/fsm)
* [go-trellis/fsm](https://github.com/go-trellis/fsm)
* ……

## 参考资料

1. <https://en.wikipedia.org/wiki/Finite-state_machine>
