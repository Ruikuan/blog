# new() 约束的陷阱

发现一篇很好的文章 [Dissecting the new() constraint in C#: a perfect example of a leaky abstraction](https://blogs.msdn.microsoft.com/seteplia/2017/02/01/dissecting-the-new-constraint-in-c-a-perfect-example-of-a-leaky-abstraction/)，在这里写一下要点。

## 问题

之前（.net framework 2.0 前时代）在项目中动态创建某个不定类型实例时，使用的是 `Activator.CreateInstance()`，虽然知道这种创建方式是通过反射实现的，性能不太理想，当时也没有太好的其他选择。后来泛型出来了，有了个 `new()` 约束，可以轻松地在代码里面 `new T()` 这样写了，心里美滋滋的，想微软还真是贴心哪，这下子不单代码更加简练了，还直接能把对象 `new` 出来，性能完美了。

```cs
public class NodeFactory
{
    public static TNode CreateNode<TNode>() 
        where TNode : Node, new()
    {
        return new TNode();
    }
}
```

没想到看了这篇文章之后才知道，原来看起来眉清目秀的 `new TNode()`，不是直接执行 `new` 指令创建出来的，而是一转身就跑去调用了 `Activator.CreateInstance<T>()`，看着靠谱的样子，实际上暗度陈仓的干活，真是让人大跌眼镜。性能比直接调用 `Activator.CreateInstance<T>` 还差。

而且这样的实现，除了性能问题，还有正确性上也存在问题：

* 万一类型的构造方法抛出异常，`new T()` 返回的异常不是构造方法抛出的那个，而是被封装到了反射 API 调用的 `TargetInvocationException` 中。虽然可以通过如下方式使用 `ExceptionDispatchInfo` 来重新抛出纠正，但代码调用方不是相当了解细节的话就容易出错。

```cs
public static T Create<T>() where T : new()
{
    try
    {
        return new T();
    }
    catch (TargetInvocationException e)
    {
        var edi = ExceptionDispatchInfo.Capture(e.InnerException);
        edi.Throw();
        // Required to avoid compiler error regarding unreachable code
        throw;
    }
}
```

* 虽然 C# 是不能创建不带参数的 `struct` 自定义构造方法的，但 clr 支持这样做，可能通过构造 `il` 或者使用其他 clr 语言构造出这样的 `struct`，由于 `Activator.CreateInstance<T>` 内部使用了 cache 来进行一定的性能优化，而这优化对于这种有不带参数的构造方法的 `struct` 存在 bug，对于这种 `struct`，`Activator.CreateInstance<T>` 只会对创建的第一个实例调用这构造方法，后面的实例都不会调用，从而造成行为方面的问题。

## 解决

那么应该如何应对这种情况？如何创建高性能而且正确的实例初始方法？

可以使用后面出来的表达式树来进行处理。表达式树是一种轻量级的代码生成方案，可以编译成 `delegate`，部分也可以编译成 `Expression<DelegateType>` 表达式。在这里我们可以使用它编译成 `delegate`。

### 版本 1

```cs
public static class FastActivator
{
    public static T CreateInstance<T>() where T : new()
    {
        return FastActivatorImpl<T>.NewFunction();
    }
 
    private static class FastActivatorImpl<T> where T : new()
    {
        // Compiler translates 'new T()' into Expression.New()
        private static readonly Expression<Func<T>> NewExpression = () => new T();
 
        // Compiling expression into the delegate
        public static readonly Func<T> NewFunction = NewExpression.Compile();
    }
}
```

`FastActivator.CreateInstance` 概念上跟 `Activator.CreateInstance` 类似，但有两点不一样：

* 它不会包装异常。
* 运行时不会依赖反射。虽然在表达式构造时有依赖，但只会发生一次。

经过基准测试，`FastActivator.CreateInstance` 比 `Activator.CreateInstance` 快 5 倍，但比通过 `Func<Node>` 直接通过方法创建实例的方式还是慢 3.5 倍。为什么慢那么多呢？因为 `Expression.Compile` 创建一个 `DynamicMethod` 并把它关联到一个匿名程序集，为了让它在一个安全的沙箱环境跑，这主要是为了跑部分可信代码的安全性考虑，但带来了运行时的开销。

可以通过将 `DynamicMethod` 的一个 `constructor` 关联到特定的模块来解决。由于使用 `Expression.Compile` 实现这个存在困难，我们可以手动 “`compile`” 我们的 `factory` 方法：

### 版本 2

```cs
public static class DynamicModuleLambdaCompiler
{
    public static Func<T> GenerateFactory<T>() where T:new()
    {
        Expression<Func<T>> expr = () => new T();
        NewExpression newExpr = (NewExpression)expr.Body;
 
        var method = new DynamicMethod(
            name: "lambda", 
            returnType: newExpr.Type,
            parameterTypes: new Type[0],
            m: typeof(DynamicModuleLambdaCompiler).Module,
            skipVisibility: true);
 
        ILGenerator ilGen = method.GetILGenerator();
        // Constructor for value types could be null
        if (newExpr.Constructor != null)
        {
            ilGen.Emit(OpCodes.Newobj, newExpr.Constructor);
        }
        else
        {
            LocalBuilder temp = ilGen.DeclareLocal(newExpr.Type);
            ilGen.Emit(OpCodes.Ldloca, temp);
            ilGen.Emit(OpCodes.Initobj, newExpr.Type);
            ilGen.Emit(OpCodes.Ldloc, temp);
        }
            
        ilGen.Emit(OpCodes.Ret);
 
        return (Func<T>)method.CreateDelegate(typeof(Func<T>));
    }
}
```

有了这个新的 helper 方法，上面的 `FastActivator` 可以修改为：

```cs
public static class FastActivator
{
    public static T CreateInstance<T>() where T : new()
    {
        return FastActivatorImpl<T>.Create();
    }
 
    private static class FastActivatorImpl<T> where T : new()
    {
        public static readonly Func<T> Create =
            DynamicModuleLambdaCompiler.GenerateFactory<T>();
    }
}
```

这个版本比上个版本快两倍，但还是比 `Func<Node>` 慢两倍。原因如下：

* 这个方法是基于泛型实现的，而调用泛型方法不会被 inline，所以多出了函数调用的开销。
* 对于引用类型的泛型，clr 在执行时需要对类型进行判断，确定类型正确，多出了检查类型的开销。而对于值类型，这个版本其实并不会慢。

### 版本 3

为了解决这个引用类型的问题，避免间接性层次的增加，我们可以将内嵌的 `FastActivatorImpl<T>` 类搬到 `FastActivator` 外面，并直接调用它：

```cs
public static class FastActivator<T> where T : new()
{
    /// <summary>
    /// Extremely fast generic factory method that returns an instance
    /// of the type <typeparam name="T"/>.
    /// </summary>
    public static readonly Func<T> Create =
        DynamicModuleLambdaCompiler.GenerateFactory<T>();
}
```

这个版本表现就相当好了，可以跟直接调用 `Func<Node>` 相比较。

## 番外

如果 JIT 支持对 `new T()` 直接生成 `new` 指令就好了。