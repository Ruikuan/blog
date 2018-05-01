# Memory<T> 使用 guideline

根据 [`Memory<T>` usage guidelines](https://gist.github.com/GrabYourPitchforks/4c3e1935fd4d9fa2831dbfcab35dffc6) 总结。

由于 `Span<T>` 跟 `Memory<T>` 的区别，以及考虑到 ownership 和 consumption，在使用 `Memory<T>` 家族时最好遵循如下 guideline。

## 1. 对于同步的 API，尽可能使用 `Span<T>` 而不是 `Memory<T>`。

因为 `Span<T>` 比 `Memory<T>` 能用于更多的情形，而且很容易能变成 `Memory<T>`。反之则不行。

## 2. 如果 buffer 是用于不可变的情形，那么应使用 `ReadOnlySpan<T>` 和 `ReadOnlyMemory<T>`。

## 3. 如果你的方法接受 `Memory<T>` 而返回 `void`，那么禁止在方法返回后再使用那个 `Memory<T>`。

错误示范：
```cs
// !!! INCORRECT IMPLEMENTATION !!!
static void Log(ReadOnlyMemory<char> message)
{
    // Run in background so that we don't block the main thread
    // while performing IO.
    Task.Run(() => {
        File.AppendText(message);
    });
}
```

在方法返回后，异步的代码还在使用 `Memory<T>`，万一返回后的代码对同样的 buffer 进行了修改，则 buffer 数据就全乱套了。

## 4. 如果你的方法接受 `Memory<T>` 而返回 Task，禁止在 `Task` 变为终结状态后继续使用这个 `Memory<T>`。

3号规则的异步版本。`Task<T>` `ValueTask<T>` 和其他类似的类型同理。

## 5. 如果你的构造方法接受 `Memory<T>` 作为参数，那么构造出来的对象也被认为作为这个 `Memory<T>` 的 consumer。

## 6. 如果你的类型有个可以设置的 `Memory<T>` 类型属性或者实例方法，那么其他实例方法也被认为是这个 `Memory<T>` 的 consumer。

 同上。

## 7. 如果你拥有一个 `IMemoryOwner<T>` 引用，那么你必须负责在某个时候 dispose 它或者将所有权转移到其他地方。（但只能干一个，不能都干。）

## 8. 如果你的 API 接受 `IMemoryOwner<T>` 参数，你正接受这个实例的 ownership。

## 9. 如果你正包装一个同步的 p/invoke 方法，应该接受 `Span<T>` 参数。

可以使用 `fixed` 关键字来 pin 住 `Span<T>` 实例。

```cs
using System.Runtime.InteropServices;

[DllImport(...)]
private static extern unsafe int ExportedMethod(byte* pbData, int cbData);

public unsafe int ManagedWrapper(Span<byte> data)
{
    fixed (byte* pbData = &MemoryMarshal.GetReference(data))
    {
        int retVal = ExportedMethod(pbData, data.Length);

        /* error checking retVal goes here */

        return retVal;
    }

    // In the above example, 'pbData' can be null; e.g., if
    // the input span is empty. If the exported method absolutely
    // requires that 'pbData' be non-null, even if 'cbData' is 0,
    // consider the following implementation.

    fixed (byte* pbData = &MemoryMarshal.GetReference(data))
    {
        byte dummy = 0;
        int retVal = ExportedMethod((pbData != null) ? pbData : &dummy, data.Length);

        /* error checking retVal goes here */

        return retVal;
    }
}
```

## 10. 如果你正包装一个异步的 p/invoke 方法，应该接受 `Memory<T>` 参数。

异步操作中不能使用 `fixed` 来 pin 住内存，取而代之，可以使用 `Memory<T>.Pin` 来 pin 住 `Memory<T>` 实例。

```cs
using System.Runtime.InteropServices;

[UnmanagedFunctionPointer(...)]
private delegate void OnCompletedCallback(IntPtr state, int result);

[DllImport(...)]
private static extern unsafe int ExportedAsyncMethod(byte* pbData, int cbData, IntPtr pState, IntPtr lpfnOnCompletedCallback);

private static readonly IntPtr _callbackPtr = GetCompletionCallbackPointer();

public unsafe Task<int> ManagedWrapperAsync(Memory<byte> data)
{
    // setup
    var tcs = new TaskCompletionSource<int>();
    var state = new MyCompletedCallbackState {
        Tcs = tcs
    };
    var pState = (IntPtr)GCHandle.Alloc();

    var memoryHandle = data.Pin();
    state.MemoryHandle = memoryHandle;

    // make the call
    int result;
    try {
        result = ExportedAsyncMethod((byte*)memoryHandle.Pointer, data.Length, pState, _callbackPtr);
    } catch {
        ((GCHandle)pState).Free(); // cleanup since callback won't be invoked
        memoryHandle.Dispose();
        throw;
    }

    if (result != PENDING)
    {
        // Operation completed synchronously; invoke callback manually
        // for result processing and cleanup.
        MyCompletedCallbackImplementation(pState, result);
    }

    return tcs.Task;
}

private static void MyCompletedCallbackImplementation(IntPtr state, int result)
{
    GCHandle handle = (GCHandle)state;
    var actualState = (MyCompletedCallbackState)state;
    handle.Free();
    actualState.MemoryHandle.Dispose();

    /* error checking result goes here */

    if (error) { actualState.Tcs.SetException(...); }
    else { actualState.Tcs.SetResult(result); }
}

private static IntPtr GetCompletionCallbackPointer()
{
    OnCompletedCallback callback = MyCompletedCallbackImplementation;
    GCHandle.Alloc(callback); // keep alive for lifetime of application
    return Marshal.GetFunctionPointerForDelegate(callback);
}

private class MyCompletedCallbackState
{
    public TaskCompletionSource<int> Tcs;
    public MemoryHandle MemoryHandle;
}
```