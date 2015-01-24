---
layout: post
title:  "面向对象设计实践指南 4 - 创建灵活的接口"
date:   2015-01-04 20:30:00
categories: ruby
tags: learningnote refactoring
author: "Victor"
---

设计要解决对象所知道的内容(职责)，对象都知道谁(依赖关系)，还要解决彼此间如何进行对话。

## 理解接口

类实现了许多方法，有些方法旨在被其他对象使用，这些方法便组成了这个类的公共接口。

## 定义接口

通用方法将主要职责暴露给细小实用的方法，而细小的方法则只在内部使用。

### 公共接口

* 暴露了其主要职责
* 期望被其他对象调用
* 不会因一时兴起而改变
* 对其他依赖它的对象来说很安全
* 在测试里被详尽记录

### 私有接口

* 要处理实现细节
* 不希望被发送到其他对象
* 可以因任何不明原因而变化
* 对其他依赖它的对象来说很不安全
* 可能在测试里被引用

### 职责、依赖关系和接口

公共方法读起来应该像是对职责的描述。公共接口是一种契约，它阐明了类的职责。

一个类应该只依赖于那些变化情况比自己还少的类。

## 找出公共接口

设计的目标是要保证未来最大的灵活性，同时只编写足以满足今天需求的代码。

### 构建意图

当我们最初构建程序的时候，总会有一些类蹦到头脑里，这是因为它们代表了应用程序里某些现实存在的名词。这叫领域对象。它们代表了现实世界里很大的、易于发现的事物。

但是设计专家会关注它们之间传递的消息。这些消息会引导你去发现其他的对象，而这些对象可能没那么明显。

### 使用时序图

时序图可用来对不同对象的排列和消息传递方案进行试验。它的价值在于明确指定了对象间传递的消息，因为对象间只应该使用公共接口进行通信。时序图便是一种用于暴露、试验并最终定义这些接口的工具。

前面的设计重点强调的是类，以及它们知道谁和知道些什么。现在我们要决定消息，并弄清要将消息发送到哪里。

我们的思维方式从 **我知道我需要这个类，那么该怎么办？** 转变为 **我需要发送这条消息，谁该响应它呢？** 注意，我们不是面向对象编程，而是面向消息编程。

**你发送消息不是因为有了对象，你有了对象是因为你要发送消息。**

### 询问 “要什么”，别告知 “如何做”

### 寻求上下文独立

类所了解到的关于其他对象的那些事情便构成了它的上下文，可以说类具有单一职责，但它期望一个上下文。对类的任何使用情况，不管是测试还是其他用途，都会要求建立其上下文。

对象所期望的上下文会直接影响到重用它的难度。具有简单上下文的对象易于测试和使用。所以最好的情况是对象与它的上下文保持完全的独立。与其他对象合作而无需知道他们是谁的技术就是指 **依赖关系注入**。

### 使用消息来发现对象

### 创建基于消息的应用程序

## 编写能展现其内在最好面的代码

请好好想一下接口。要有意识的创建它们。定义应用程序，并决定其未来的是接口，而非测试用例或代码。

### 创建显示接口

每当你创建类时，请声明它的接口。公共接口里的方法应该满足以下几点：

* 被明确标示
* 多于 **做什么** 有关，少于 **怎么做** 有关
* 尽可能让这些名字都稳定不变
* 将散列表作为选项参数

另外也请时刻注意以下规则：

* 善用其他类的公共接口
* 避免依赖私有接口
* 最小化上下文

要创建这样的公共方法：它允许发送者在不了解类时如何实现其行为的情况下，也可以获得它所要的内容。

## 迪米特法则

迪米特法则会限制向某个方法发送消息的对象集合。它禁止这样的做法：将某条消息通过第二个不同类型的对象转发给第三个对象。

### 定义迪米特法则

也可以理解成：只能与近邻传递消息或只能使用一个小圆点。

```ruby
# bad
customer.bicycle.wheel.tire
customer.bicycle.wheel.rotate
# good
hash.keys.sort.join(", ")
```

注意上面的例子，事实上评价的方法不是依靠统计小圆点个数，而是检查中间对象的类型是否变化。

```ruby
hash.keys #=> Enumerable
hash.keys.sort #=> Enumerable
hash.keys.sort.join(", ") #=> String
```

### 如何避免违规

一种常见的办法是利用委托来避开这些小圆点。Ruby 提供 ```delegate.rb``` 和 ```forwardable.rb``` 而 Rails 提供 ```delegate``` 方法。

但是请注意 **用委托来隐藏紧耦合** 和 **对代码进行解耦** 是两码事。

### 听从迪米特法则

迪米特法则是要告诉你某件事，而不是让你大量使用委托。

当你的设计思想过度受到现有对象的影响时，就会出现 ``customer.bicycle.wheel.tire`` 这样的代码。因为你熟悉那些接口，所以你自然而然的为了获得远距离的行为而使用这长长的消息链。



