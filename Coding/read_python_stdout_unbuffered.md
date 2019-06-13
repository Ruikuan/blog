# 使用 Process 启动 python 进程并顺利读取标准输出流

dotnet 中使用 `Process` 来启用子进程是很常见的操作，还可以配置为重定向输入输出流，对操作命令行程序很方便。一般的使用代码如下：

```cs
static void Main(string[] args)
{
    ProcessStartInfo startInfo = new ProcessStartInfo("app.exe", "args");
    startInfo.UseShellExecute = false;
    startInfo.RedirectStandardOutput = true;
    startInfo.RedirectStandardError = true;

    Process process = new Process() { StartInfo = startInfo };
    process.EnableRaisingEvents = true;
    process.OutputDataReceived += Process_OutputDataReceived;
    process.ErrorDataReceived += Process_ErrorDataReceived;
    process.Exited += Process_Exited;
    process.Start();
    process.BeginOutputReadLine();
    process.BeginErrorReadLine();
    process.WaitForExit();
}

private static void Process_Exited(object sender, EventArgs e)
{
    Console.WriteLine($"ExitCode {((Process)sender).ExitCode}");
}

private static void Process_ErrorDataReceived(object sender, DataReceivedEventArgs e)
{
    Console.WriteLine($"Error: {e.Data}");
}

private static void Process_OutputDataReceived(object sender, DataReceivedEventArgs e)
{
    Console.WriteLine($"{e.Data}");
}
```

对于调用普通的程序，这样就已经足够了。但用来调用 python 程序就很奇怪。（测试环境为 python 3.7 ）

```py
import time
for i in range(100):
    print('message %d ' % i)
    time.sleep(0.2)
```

对于上面这个 python 脚本，直接在 shell 中运行的话，会接连输出一串文本行，每 0.2 秒一次。但通过下面的脚本去调用的话，则等待了若干秒（100 * 0.2 = 20）之后，才会把结果一股脑输出。

```cs
ProcessStartInfo startInfo = new ProcessStartInfo("python", "a.py");
```

查了资料，原来 python 会默认 buffer 标准输出流，导致这个现象，解决方法很简单，加一个 `-u` 参数即可。

```cs
ProcessStartInfo startInfo = new ProcessStartInfo("python", "-u a.py");
```

命令行说明：
```
-u
Force the stdout and stderr streams to be unbuffered. This option has no effect on the stdin stream.
```

> [1. Command line and environment](https://docs.python.org/3/using/cmdline.html)
