---
title: AutoMapper 升级指南
date: 2018-11-09 13:11:13
category:
 - AutoMapper
tags:
---

# 初始化

您现在必须使用 `Mapper.Initialize` 或 `new MapperConfiguration()`来初始化 `AutoMapper`。如果您希望保持静态使用，请使用 `Mapper.Initialize`。

如果你有很多的 `Mapper.CreateMap` 调用，把它们移动到一个 `Profile`，或者 `Mapper.Initialize`，在启动时调用一次。

例如在这里看到。

# Profiles

不要覆盖 Configure 方法，而是直接通过构造函数进行配置:

```c#
public class MappingProfile : Profile {
    public MappingProfile() {
        CreateMap<Foo, Bar>();
        RecognizePrefix("m_");
    }
}
```

# IgnoreAllNonExisting 扩展

一个流行的 Stack Overflow 文章介绍了忽略目标类型上的所有不存在的成员的想法。 它使用配置 API 中不再存在的东西。 该功能实际上仅用于配置验证。

在 5.0 中，您可以使用在成员列表枚举中传递的 `ReverseMap` 或 `CreateMap` 来针对源成员（或没有成员）进行验证。 任何你有这个 `IgnoreAllNonExisting` 扩展名的地方，都可以使用 `CreateMap` 重载来对源或者非成员进行验证：

```c#
cfg.CreateMap<ProductDto, Product>(MemberList.None);
```

# 解决上下文事物

`ResolutionContext` 用于捕获大量信息，源和目标值，以及分层父模型。 对于源/目标值，所有接口（值解析器和类型转换器）以及配置选项现在包括源/目标值，以及源/目标成员（如果适用）。

如果您试图访问模型中的某个父对象，则需要将这些关系添加到模型中，并通过这些关系访问它们，而不是通过 AutoMapper 的层次结构访问它们。 ResolutionContext 由于性能和完整性的原因而被削减。

# 值解析器

值解析器的签名已更改为允许访问源/目标模型。 另外，基类已经不再用于接口。 对于没有成员重定向的值解析器，接口现在是：

```c#
public interface IValueResolver<in TSource, in TDestination, TDestMember>
{
    TDestMember Resolve(TSource source, TDestination destination, TDestMember destMember, ResolutionContext context);
}
```

您现在可以访问此解析器针对的源模型，目标模型和目标成员。

如果您正在使用 `ResolveUsing` 并传递 `FromMember` 配置，则现在这是一个新的解析器接口：

```c#
public interface IMemberValueResolver<in TSource, in TDestination, in TSourceMember, TDestMember>
{
    TDestMember Resolve(TSource source, TDestination destination, TSourceMember sourceMember, TDestMember destMember, ResolutionContext context);
}
```

这现在被直接配置为

```c#
ForMember(dest => dest.Foo, opt => opt.ResolveUsing<MyCustomResolver, string>(src => src.Bar)
```

# 类型转换器

类型转换器的基类现在已经变成了接受源和目标对象并返回目标对象的单一接口：

```c#
public interface ITypeConverter<in TSource, TDestination>
{
    TDestination Convert(TSource source, TDestination destination, ResolutionContext context);
}
```

# 循环引用

以前，`AutoMapper` 可以通过跟踪所映射的内容来处理循环引用，并且在每个映射上检查源/目标对象的本地散列表，以查看该项是否已被映射。 事实证明，这种跟踪是非常昂贵的，你需要选择使用 `PreserveReferences` 循环映射工作。 或者，您可以配置 `MaxDepth`：

```c#
// 自引用映射
cfg.CreateMap<Category, CategoryDto>().MaxDepth(3);

// users 和 groups 之间的循环引用
cfg.CreateMap<User, UserDto>().PreserveReferences();
```

从 6.1.0 开始，只要静态检测到递归，就可以在配置时自动设置 `PreserveReferences`。 如果在您的情况下没有发生这种情况，请使用完整的 repro 开启一个问题，我们会研究它。

# UseDestinationValue

`UseDestinationValue` 告诉 `AutoMapper` 不要为某个成员创建一个新的对象，而是使用目标对象的现有属性。 它以前是默认的。 考虑这是否适用于你的情况。 [检查最近的问题](https://github.com/AutoMapper/AutoMapper/search?o=desc&q=UseDestinationValue&s=created&type=Issues&utf8=%E2%9C%93)

```c#
cfg.CreateMap<Source, Destination>()
   .ForMember(d => d.Child, opt => opt.UseDestinationValue());
```
