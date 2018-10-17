---
layout: post
title: C#中跨线程安全操控控件
tags:
 - 原创
 - C# 
 - 多线程
 - 控件
date: 2018-07-24 10:00 # 强制回溯到正确的发布时间
---

> 在C#的多线程编程中，如果产生“线程间操作无效：从不是创建控件XXXX的线程访问它”的异常，那是因为默认情况下，在Windows应用程序中，.NET Framework不允许在一个线程中直接操作另一个线程中的控件(因为访问Windows窗体控件本质上不是线程安全的)。微软为了线程安全，窗体上的控件只能通过创建控件的线程来操作控件的数据，也就是只能是UI线程来操作窗体上的控件！

<!-- more -->

{% asset_img JLoeve_2018-07-24_20-23-42.png 线程间操作无效 %}

## 第一种做法

直接关闭关闭线程安全检查，**这种做法极其不负责任，请避免使用此方法**

```CSharp
Control.CheckForIllegalCrossThreadCalls = false;  // 关闭线程安全检查
```

## 第二种做法

使用委托，这种方法是在我的微博中介绍过的，不过实际写起来还是有些麻烦

```CSharp
public delegate void UpdateUI(string mes);

public void UpdateText(string text)
{
    if (label1.InvokeRequired)
    {
        var UU = new UpdateUI(UpdateText);
        Invoke(UU,text);
        //return;
    }
    else  // 上面使用了return的话，那么这个else则可以去掉
        label1.Text = text;
}

private void Form1_Load(object sender, EventArgs e)
{
    var t = new Thread(new ThreadStart(() =>
    {
        while (true)
            UpdateText(DateTime.Now.ToLongTimeString());
    }));
    t.Start();
    /*
    当你关闭主窗口时，UI线程同时被终止，但是新开启的线程仍然不会终止，
    新线程仍然将不断尝试更新已经被销毁的控件上的文本内容，
    由此造成的结果就是```Invoke(UU,text);```
    这句将会在调试模式下抛出```System.ObjectDisposedException```异常（直接运行则无碍）。
    */
    FormClosing += (o, e2) => t.Abort();   // 记得在窗口将要被关闭时终止新开启的线程
}
```

## 第三种做法

使用控件的Invoke方法或BeginInvoke方法

### Invoke

```CSharp
private void Form1_Load(object sender, EventArgs e)
{

    var t = new Thread(new ThreadStart(() =>
    {
        while (true)
            label1.Invoke(new Action(() => label1.Text = DateTime.Now.ToLongTimeString()));

    }));
    t.Start();
    FormClosing += (o, e2) => t.Abort();
}
```

显然这种写法相比使用委托要更为精简

### BeginInvoke

```CSharp
private void Form1_Load(object sender, EventArgs e)
{

    var t = new Thread(new ThreadStart(() =>
    {
        while (true)
        {
            // 这里是异步执行的，即不等待UI线程执行完毕就返回
            label1.BeginInvoke(new Action(() => label1.Text = DateTime.Now.ToLongTimeString()));  
            Thread.Sleep(1000);   // 这里需要挂起线程，如果不挂起，那么系统内存将很快耗尽（而且UI线程的消息队列将被塞得满满当当）
        }
    }));
    t.Start();
    FormClosing += (o, e2) => t.Abort();
}
```

### 关于Invoke和BeginInvoke方法的区别请参考[这篇文章](https://www.cnblogs.com/fuchongjundream/p/3939298.html)

这里我转载一段作为重点：

Invoke或者BeginInvoke方法都需要一个委托对象作为参数。委托类似于回调函数的地址，因此调用者通过这两个方法就可以把需要调用的函数地址封送给界面线程。这些方法里面如果包含了更改控件状态的代码，那么由于最终执行这个方法的是界面线程，从而避免了竞争条件，避免了不可预料的问题。如果其它线程直接操作界面线程所属的控件，那么将会产生竞争条件，造成不可预料的结果。

使用Invoke完成一个委托方法的封送，就类似于使用SendMessage方法来给界面线程发送消息，是一个同步方法。也就是说在Invoke封送的方法被执行完毕前，Invoke方法不会返回，从而调用者线程将被阻塞。

使用BeginInvoke方法封送一个委托方法，类似于使用PostMessage进行通信，这是一个异步方法。也就是该方法封送完毕后马上返回，不会等待委托方法的执行结束，调用者线程将不会被阻塞。但是调用者也可以使用EndInvoke方法或者其它类似WaitHandle机制等待异步操作的完成。

但是在内部实现上，Invoke和BeginInvoke都是用了PostMessage方法，从而避免了SendMessage带来的问题。而Invoke方法的同步阻塞是靠WaitHandle机制来完成的。

## 第四种做法

使用 BackgroundWorker，这个组件常常用于给UI线程中的进度条报告进度

```CSharp
private void Form1_Load(object sender, EventArgs e)
{
    backgroundWorker1.DoWork += (o, e2) =>
    {
        // 注意在 DoWork 事件中不能直接调用UI控件，这个方法的执行是在另外的线程中进行的
        while (true)
        {
            // 注意，使用此方法报告进度时应将backgroundWorker1的属性WorkerReportsProgress设置为true
            backgroundWorker1.ReportProgress(0, DateTime.Now.ToLongTimeString());  
            Thread.Sleep(1000); // 还是得加这句延时，严重怀疑上面调用的方法也是异步调用的
        }
    };
    backgroundWorker1.ProgressChanged += (o, e3) =>
    {
        var text = e3.UserState as String;
        label1.Text = text;
    };
    backgroundWorker1.RunWorkerAsync();
}
```

