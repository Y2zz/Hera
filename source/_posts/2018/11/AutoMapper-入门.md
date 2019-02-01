---
title: AutoMapper 入门
date: 2018-11-09 12:54:10
category:
 - AutoMapper
tags:
 - AutoMapper
 - 映射
 - DTO
---

# 什么是 AutoMapper？

AutoMapper 是一个对象与对象的映射器。对象-对象映射通过将一种类型的输入对象转换为不同类型的输出对象来工作。让 AutoMapper 感兴趣的是，它提供了一些有趣的约定，从而弄清楚如何将类型 A 映射到类型 B.只要类型 B 遵循 AutoMapper 建立的约定，映射两种类型几乎就是零配置。

# 为什么使用 AutoMapper？

映射代码很无聊。测试映射代码更无聊。 AutoMapper 提供了简单的类型配置，以及简单的映射测试。真正的问题可能是“为什么要使用对象映射？”，映射可以在应用程序的许多地方发生，但大多数情况下可以在层之间的边界（如 UI/Domain 层或 Service/Domain 层）之间进行。一层的关注经常与另一层的关注相冲突，所以对象-对象映射导致隔离模型，其中每层的关注只能影响该层中的类型。

# 我如何使用 AutoMapper？

首先，您需要使用源类型和目标类型。目标类型的设计可能会受到其所在层的影响，但只要成员的名称与源类型的成员相匹配，AutoMapper 就可以发挥最佳效果。如果您有一个名为`FirstName`的源成员，则会自动将其映射到名为`FirstName`的目标成员。 AutoMapper 也支持拼合。

将源映射到目标时，AutoMapper 将忽略空引用异常。这是设计。如果您不喜欢这种方法，可以将 AutoMapper 的方法与自定义值解析器结合使用（如果需要的话）。

一旦你有了你的类型，你可以使用`MapperConfiguration`或静态`Mapper`实例和`CreateMap`为这两种类型创建一个映射。通常每个`AppDomain`只需要一个`MapperConfiguration`实例，并应在启动过程中实例化。或者，您可以使用 `Mapper.Initialize`（初始设置的更多示例请参阅 [Static-and-Instance-API](https://automapper.readthedocs.io/en/latest/Static-and-Instance-API.html)。

```c#
Mapper.Initialize(cfg => cfg.CreateMap<Order, OrderDto>());
//or
var config = new MapperConfiguration(cfg => cfg.CreateMap<Order, OrderDto>());
```

左边的类型是源类型，右边的类型是目标类型。 要执行映射，请使用静态或实例映射器方法，具体取决于静态或实例初始化：

```c#
var mapper = config.CreateMapper();
// or
var mapper = new Mapper(config);
OrderDto dto = mapper.Map<OrderDto>(order);
// or
OrderDto dto = Mapper.Map<OrderDto>(order);
```

大多数应用程序可以使用依赖注入来注入创建的 IMapper 实例。

AutoMapper 也有这些方法的非通用版本，对于那些在编译时可能不知道类型的情况。

# 我在哪里配置 AutoMapper？

如果您使用的是静态`Mapper`方法，则每个`AppDomain`只能进行一次配置。 这意味着放置配置代码的最佳位置是在应用程序启动时，例如 ASP.NET 应用程序的`Global.asax`文件。 通常，配置引导程序类位于其自己的类中，并且此引导程序类是从启动方法调用的。 引导程序类应该调用`Mapper.Initialize`来配置类型映射。

# 我如何测试我的映射？

为了测试你的映射，你需要创建一个测试来做两件事情：

> 调用你的引导类来创建所有的映射
> 调用 MapperConfiguration.AssertConfigurationIsValid

这是一个例子：

```c#
var config = AutoMapperConfiguration.Configure();
config.AssertConfigurationIsValid();
```
