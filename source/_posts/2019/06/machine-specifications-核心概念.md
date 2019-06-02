---
title: machine.specifications - 核心概念
date: 2019-06-02 13:02:03
category:
tags: BDD, MSpec
---

MSpec 被称为 “上下文/规范” 测试框架，because of 作为 “语法” 用于描述和编码测试或 “specs”，语法读起来就像这样

> 当系统处于这种状态并且发生某种动作时，它应该这样做或处于某种最终状态。

你应该能够在这里看到传统[Arrange-Act-Assert](https://docs.microsoft.com/en-us/visualstudio/test/unit-test-basics#write-your-tests)模式的组件。为了支持可读性并尽可能多的移除“噪音”，MSpec 避开了传统的测试构造属性方法模式。它使用自定义 .NET 委托，你可以指定匿名方法，并要求你按照特定的约定命名它们。

# Subject

Subject 特性是 Spec 类的一部分。它描述了“上下文”，它可以是被测试的文字类型或更广泛的描述。该 Subject 不是必须的，但最好添加它。此外，该特性允许 ReSharper 或 Rider 检测上下文，以便委托成员不会被视为未使用。

这个类命名约定是使用 Sentence_snake_case 并以单词“When”开头。

```c#
[Subject("Authentication")]                           // 说明
[Subject(typeof(SecurityService))]                    // 被测试的类型
[Subject(typeof(SecurityService), "Authentication")]  // 或一个组合!
class When_authenticating_a_user { ... }              // 记住：你只能使用一个 Subject Attribute!
```

# Tags

Tags 特性用于组织 Spec 类，以便在测试运行时包含或排除。你可以通过将测试标记为“Slow”来识别数据库中的测试，或者通过标记“AcceptanceTest”来识别专题报告中的测试。

Tags 可用于在 Spec 运行期间包含或排除某些上下文，请参阅 [命令行](https://github.com/machine/machine.specifications/wiki/Command-line)

```C#
[Tags("RegressionTest")]  // 此特性通过 params string[] 参数支持任意数量的标记！
[Subject(typeof(SecurityService), "Authentication")]
class When_authenticating_a_user { ... }
```

# Establish

Establish 是 Spec 类的 “Arrange” 部分，Establish 只运行一次，所以不应该在此改变任何状态。

```C#
[Subject("Authentication")]
class When_authenticating_a_new_user
{
    static SecurityService subject;

    Establish context = () =>
    {
        // ... 任何 mock, 初始化 subject, 或其他设置 ...
        subject = new SecurityService(foo, bar);
    };
}
```

为了更好的控制，Establish 也可以与继承类和嵌套类一起使用，请参阅 [继承](https://github.com/machine/machine.specifications/wiki/Inheritance)

# Cleanup

清理 Establish ，在所有 Spec 运行后会调用一次。

```C#
[Subject("Authentication")]
class When_authenticating_a_user
{
    static SecurityService subject;

    Establish context = () =>
        subject = new SecurityService(foo, bar);

    Cleanup after = () =>
        subject.Dispose();
}
```

# Because

Because 委托是 Spec 类的“Act”部分。它应该是这个上下文的唯一动作，是唯一改变状态的部分， 大多数情况这里只有一行代码。

```C#
[Subject("Authentication")]
class When_authenticating_a_user
{
    static SecurityService subject;

    Establish context = () =>
        subject = new SecurityService(foo, bar);

    Because of = () =>
        subject.Authenticate("username", "password");
}

```

如果你有一个多行 Because 语句，你可能需要确定哪些才是实际设置并将它们移动到 Establish 中。或者，你的 Spec 可能涉及太多的上下文，需要拆分，需要重构待测试的 Subject。

# It

It 委托是 Spec 类的“断言”部分。 它可能在 Spec 类中出现一次或多次。每次出现应该包含一个断言，因此意图和失败报告将非常明确。就像 Because 语句一样，It 语句通常也是一行代码。

```C#
[Subject("Authentication")]
class When_authenticating_an_admin_user
{
    static SecurityService subject;
    static UserToken user_token;

    Establish context = () =>
        subject = new SecurityService(foo, bar);

    Because of = () =>
        user_token = subject.Authenticate("username", "password");

    It should_indicate_the_users_role = () =>
        user_token.Role.ShouldEqual(Roles.Admin);

    It should_have_a_unique_session_id = () =>
        user_token.SessionId.ShouldNotBeNull();
}
```

没有被赋值的`It`语句在测试运行器中将被标记为“未实现”。你可能会发现像这样“stubbing” 你的断言可以帮助你实践 TDD。

```C#
It should_list_your_authorized_actions;
```

# Assertion

正如你在上面看到的，`It`使用了`Should`断言库的扩展方法 `ShouldEqual`、`ShouldNotBeNull`。
对于复杂的自定义对象或领域概念，应该使用自己的断言扩展方法，这是一种很好的做法。

# Ignore

任何测试框架都允许你忽略不完整或失败的测试，MSpec 为此提供了 Ignore 属性。只需要描述说明忽略的原因。

```C#
[Ignore("We are switching out the session ID factory for a better implementation")]
It should_have_a_unique_session_id = () =>
    user_token.SessionId.ShouldNotBeNull();
```

# Catch

捕获测试抛出的异常信息，你应该使用`catch`。防止测试运行失败并且在断言中检查异常的预期属性。

```C#
[Subject("Authentication")]
class When_authenticating_a_user_fails_due_to_bad_credentials
{
    static SecurityService subject;
    static Exception exception;

    Establish context = () =>
        subject = new SecurityService(foo, bar);

    Because of = () =>
        exception = Catch.Exception(() => subject.Authenticate("username", "password"));

    It should_fail = () =>
        exception.ShouldBeOfExactType<AuthenticationFailedException>();

    It should_have_a_specific_reason = () =>
        exception.Message.ShouldContain("credentials");
}
```

英文原文：<https://github.com/machine/machine.specifications/wiki/Core-Concepts>
