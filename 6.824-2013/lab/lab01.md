# 6.824 Lab1:Lock Server

## Introduction

你的任务是创建一个lock service，当一台server发生故障时，它还能正常工作。
lock service中维持了一组locks的状态。

你的lock service 应该运行(replicated)在两台servers上，一台作为primary，另外一台作为backup。

最关键的地方是，backup需要维持和primary一样的状态。

这个系统唯一需要tolerate的failure是"a single fail-stop server failure"。

随后的lab会需要tolerate更多的failures。

## 各种情况

1. primary和backup正常运行。client把request发给primary, primary把request转发给backup。
2. primary当机，backup正常运行。client把request发给primary报错，client再把request发给backup。
3. primary把request转发给backup之后，primary当机。client把request重发给backup。


## 思考

1. 因为backup需要和primary保持同样的状态，所以需要一个同步机制(replicated)。
2. 如何处理client重新发送的相同request？
3. locks的状态是由什么确定的？是多个client发起的request的次序决定的？

## 实现

1. 需要一个容器，维护locks的状态。
2. 当primary接收到一个request时，它会转发给backup。
3. 需要一个容器，里面有各个client发送的最后一个request和reply。当primary和backup收到相同的request时，它会回复以前的reply。
4. locks的状态是由收到request的次序决定的，是由外面看到的request决定的，需要把lock service看成一个黑箱子。

## 总结

设计系统的思路：

1. 定义功能，设计架构,定义状态(状态包括哪些key和value),定义能改变状态的动作。
2. 定义能够容忍的failure
3. 列出各种failure的情况
4. 找出各种failure情况的解决办法
