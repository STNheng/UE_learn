# Unity协程

## 协程

* Coroutine
	Unity协程是一个能暂停执行（暂停后立即返回），直到中断指令（yield instruction）完成后继续执行的函数。

* StartCoroutine
	一个协程在执行过程中，可以在任意位置使用yield语句暂停。yield的返回值控制何时恢复程序向下执行。协程在对象自有帧执行过程中表现优秀，性能上几乎没有更多开销。StartCorotine函数是立刻返回的，但是yield可以延迟结果。

## 包含类和常用接口

* 类
	public class MonoBehaviour
* 接口
	public Coroutine StartCoroutine(IEnumerator routine)//启动协程
	public Coroutine StartCoroutine(string methodName)//通过函数名字启动协程
	public void StopAllCoroutine()//关闭协程
	public void StopCoroutine(string methodName)//通过函数名关闭协程，对应StartCoroutine(string methodName)启动协程
	public void StopCoroutine(Coroutine routine)//关闭协程，需要传入StartCoroutine的返回值。

## 关注事项

* 依赖MonoBehaviour，继承MonoBehaviour可以使用完整的方法。非MonoBehaviour对象可以借助其他MonoBehaviour对象启动协程，但是无法使用Corotine StartCoroutine(string methodName)。

	```c#
	public class Test : MonoBehaviour
	{
	    public Coroutine coroutine=null;
	    void Start()
	    {
	        //Coroutine StartCoroutine(IEnumerator routine)
	        coroutine=StartCoroutine(Func());
	        //Coroutine StartCoroutine(string methodName)
	        coroutine=StartCoroutine("Func");
	    }
	    public IEnumerator Func()
	    {
	        Debug.Log("Func()!");
	        yield return null;
	    }
	}
	```

* yield语句：协程函数内容必须包含yield
	yield return null/N/“NNN”;这会导致协程会停止后续代码执行，把他们放到下一帧再执行；
	yield return new WaitForEndOfFrame();等待直到当前帧结束，这在Unity渲染每一个摄像机和GUI之后，在屏幕显示该帧之前；
	yield return new WaitForFixedUpdate();等待，直到下一个固定帧率更新函数；
	yield return new WaitForSeconds(N);N秒后返回执行；
	yield return new WWW(url);等待www请求完成；
	yield return StartCoroutine();等这个新的协程完成在继续
	yield break;直接退出，不执行后续代码；
	yield return Application.LoadLevelAsync(levelName);等待levelName被加载后在执行；
	yield return Resources.UnloadUnusedAssets();等待unload完成。

	![参考unity脚本生命周期](D:\电子书\monobehaviour_flowchart.svg)

	```c#
	public class Test : MonoBehaviour
	{
	    public Coroutine coroutine = null;
	    void Start()
	    {
	        Debug.Log("Start:"+Time.time);
	        coroutine = StartCoroutine(Func());
	        
	
	    }
	    void FixedUpdate()
	    {
	        Debug.Log("FixedUpdate:" + Time.time);
	    }
	    void Update()
	    {
	        Debug.Log("Update:" + Time.time);
	    }
	    
	    public IEnumerator Func()
	    {
	        
	        yield return new WaitForFixedUpdate();
	        Debug.Log("Func FixedUpdate:" + Time.time);
	        yield return new WaitForEndOfFrame();
	        Debug.Log("Func EndOfFrame:" + Time.time);
	        yield return null;
	        Debug.Log("Func Next Frame:" + Time.time);
	    }
	
	}
	```

	

* Coroutine的释放
	使用StartCoroutine函数会返回Coroutine，我们可以用一个变量接受返回值的引用。在需要停止时释放这个Coroutine(StopAllCoroutine/StopCoroutine(Coroutine coroutine)/StopCoroutine(Coroutine routine))。

* 对象的隐藏不会停止协程的执行，只有对象的销毁才会停止协程。协程不是多线程，它在主线程中运行，避免栓塞操作的逻辑，协程的嵌套逻辑会带来更多的内存资源开销，尽量减少协程的使用。

## 协程源码分析

编译器在处理协程的时候会自动生成一些代码，这些代码才是协程的“本体”。
我们首先定义一个协程函数。

``` c#
public IEnumerator MyCoroutine()
{
    yield return new WaitForEndOfFrame();
    Debug.Log("Frame End");
    yield return null;
    Debug.Log("null");
    yield return new WaitForSeconds(1);
    Debug.Log("after 1s");
}
```

对生成的dll反编译，会发现编译器自动生成一个类。

```c#
private sealed class<MyCoroutine>c_Iterator1:IEnumerator<object>,IEnumerator,IDisposable
{
    internal object $current;
    internal int &PC;
    public void Dispose();
	public bool MoveNext();
	public void Reset();
	object IEnumerator<object>.Current;
	object IEnumerator.Current;
}
```

StartCoroutine函数接收的参数实际上是编译器根据协程函数生成的一个类对象，该对象有两个变量：current和PC，比较重要的函数有MoveNext函数和get_Current函数。PC表示协程函数当前执行到的位置，MoveNext函数则根据PC选择分支执行(switch(PC))，yield return语句会把协程函数分为若干段，并通过PC的值加以区分。

Unity每一帧都会调用一次MoveNext，具体调用的时间和yield return语句返回的类型有关

## 协程优缺点

优点

* **简化异步编程**：协程可以让你用同步的方式编写异步的代码，使得代码更易于理解和维护。
* **非阻塞**：协程可以在等待某些操作（如I/O操作、网络请求、延迟等）完成时不阻塞游戏的主线程，从而保持游戏的流畅性。
* **灵活性**：协程可以在任何时间点暂停和恢复，这使得它在处理一些复杂的逻辑或状态时非常灵活。

缺点

* **性能**：协程的开销比普通函数调用要大，特别是在创建和销毁协程时。如果过度使用或不正确使用协程，可能会导致性能问题。
* **错误处理**：协程的错误处理比普通函数更复杂。如果一个协程中抛出了异常，Unity不会停止游戏，而是只会在控制台输出错误信息。这可能会使得一些错误难以被发现和调试。
* **控制流**：虽然协程可以让你用同步的方式编写异步的代码，但它也使得控制流变得更复杂。在一些复杂的情况下，协程的控制流可能会变得难以理解和管理。

### 补充

在MonoBehaviour类定义中，存在以下方法。
``` c#
public Coroutine StartCoroutine(string name);
public Coroutine StartCoroutine(IEnumerator routine);
```

他们最终分别调用MonoBehaviour里面的函数。
``` c#
[MethodImpl(MethodImplOptions.InternalCall)]
private extern Coroutine StartCoroutineManaged(string methodName, object value);

[MethodImpl(MethodImplOptions.InternalCall)]
private extern Coroutine StartCoroutineManaged2(IEnumerator enumerator);
```

这两个函数实在外部定义的。[MethodImpl(MethodImplOptions.InternalCall)]是一个特性，它指示该方法的实现是由运行时提供的。
所以只有MonoBehaviour对象才有启动协程的能力。

此外MonoBehaviour还具有
``` c#
public void Invoke(string methodName,float time)
```

此函数表示在time秒后调用methodName方法。但是官方：为了更好的性能和可维护性，请使用协程。

