---
title: "迭代的认识"
author: "patrick.unicorn"
date: "Friday, December 05, 2014"
output: pdf_document
---

迭代的认识dd
====

**声明：** 所写均为个人阅读所思所想，限于知识层次和结构，难免有错误遗漏之处，请批判阅读。探讨可循：<mr.geek.ma@foxmail.com>

* * *

## Concept(概念)

迭代即谓“遍历”，它是对“集合”这一数学概念中的**元素遍历动作**的抽象指称。迭代器模式也是GOF23中模式中的一种。迭代器的**全部意义在于通过迭代器将遍历行为从数据结构(对象)中抽象(离)出来。**通常我们都习惯使用平台类库自带的集合类，这些集合类均自实现了迭代功能，通过特定的语法结构即可迭代集合对象，如Java中的`for(ItemType item : collections)`或C#中的`foreach(ItemType item in collections)`。直观上而言，集合迭代看起来非常“简洁”，但是，“简洁”不代表“简单”。如果自己尝试实现过迭代器，那么，你一定会有非常深刻的体会。在下文的叙述中，我会试着从Java或C#中的迭代器相关实现中，解析其原理和用途。

<!--more-->

## Iteration & Iterator

#### Why Need Two Similar Interface(为什么需要两个相似的迭代接口)

在Java和C#中关于迭代的接口分别如下：

+ Java: `Iterable` `Iterator`

```
public interface Iterable<T> {

    Iterator<T> iterator();
}

public interface Iterator<E> {

    boolean hasNext();

    E next();

    void remove();
}
```

+ C#: `IEnumerable` `IEnumerator`

```
public interface IEnumerable<T> {

    IEnumerator<T> GetEnumerator();
}

public interface IEnumerator<E> {

    E Current { get; }

    bool MoveNext();

    void Reset();
}
```

一个直观的疑问是“为什么不是一个接口而是两个”，下面以Java为例进行具体的分析：

从类设计上看，如果一个对象需要**被迭代**，那么它的类需要**实现Iterable**；而该迭代对象内部的**迭代器组件**则需**实现Iterator**。也就是说Iterator接口是Iterable接口的“子元素”(从两个接口的方法中可以明显看出这点)，其功能接口在理论上也可以合并到Iterable接口中。按着这个思路，我们将两个接口合并之后形成新的迭代接口：

```
public interface NewIterable<T> {

    NewIterable<T> iterator();

    boolean hasNext();

    T next();

    void remove();
}
```

实现该新接口等价于同时实现`Iterable`和`Iterator`两者。下面基于该新接口实现一个迭代类：

```
public final class Iteration implements NewIterable<Object> {

    private String[] arrays = new String[20];

    @Override
    public NewIterable<Object> iterator() {
        return this;
    }

    @Override
    public boolean hasNext() {
        // TODO Auto-generated method stub
        return false;
    }

    @Override
    public Object next() {
        // TODO Auto-generated method stub
        return null;
    }

    @Override
    public void remove() {
        // TODO Auto-generated method stub
        
    }
}
```

看上去这样合并两个接口也是可以的，但是，关键的问题在于实现者中的`Iterator<T> iterator()`方法只能返回`this`，而这会导致一些实现困难，比如如下的多层自循环结构：

```
Iteration iObject = new Iteration();

for (object i : iObject) {
    for (object i : iObject) {
        ...
    }
}
```

上面的嵌套自循环中，内存和外层的迭代器均是被迭代对象本身，共享了对内部状态变量，因此，会不能产生预期的功能。导致问题的根本原因在于**被迭代对象**本身充当了**自身的迭代器**，因此，两个接口(`Iterable`、`Iterator`)的设计，就是为了对“被迭代对象”和“迭代器”两种对象职责的进行显示区分。

#### Why Need Yield Keyword In C#(为什么C#中引入Yield关键字)

引入`Yield`关键字的唯一原因就是**简化迭代器实现**。下面以C#为例展示了使用yield和不适用yeild的迭代器的不同实现：

+ 不用yield

```
public class IterableObject : IEnumerable<object> {

    private object element0;

    private object element1;

    private object element2;

    public const int Len = 3;

    public IterableObject(object e0, object e1, object e2) {
        this.element0 = e0;
        this.element1 = e1;
        this.element2 = e2;
    }

    public object Element0 {
        get { return element0; }
    }

    public object Element1 {
        get { return element1; }
    }

    public object Element2 {
        get { return element2; }
    }

    public IEnumerator<object> GetEnumerator() {
        return new Iterator(this);
    }

    IEnumerator IEnumerable.GetEnumerator() {
        return new Iterator(this);
    }

}

class Iterator : IEnumerator<object> {

    private IterableObject actualObj;

    private int currentIndex = -1;

    public Iterator(IterableObject actualObj) {
        this.actualObj = actualObj;
    }

    public object Current {
        get {
            switch (currentIndex) {
                case 0:
                    return actualObj.Element0;
                case 1:
                    return actualObj.Element1;
                case 2:
                    return actualObj.Element2;
                default:
                    throw new InvalidOperationException();
            }
        }
    }

    public bool MoveNext() {
        if (currentIndex != IterableObject.Len) {
            ++currentIndex;
        }
        return currentIndex < IterableObject.Len;
    }

    public void Reset() {
        currentIndex = -1;
    }

    public void Dispose() {
        actualObj = null;
    }
}
```

上面的实现很简陋，为保持简单也并没有考虑线程安全性。

**注：从OOP设计原则而言，更好的选择是将`Iterator`类作为`IterableObject`，这样做的直接优点是可以对外隐藏`IterableObject`内部的数据元素(`element0`、`element1`、 `element2`)。**

+ 使用yield

```
class IterableObjectWithYield : IEnumerable<object> {

    private object element0;

    private object element1;

    private object element2;

    public const int Len = 3;

    public IterableObjectWithYield(object e0, object e1, object e2) {
        this.element0 = e0;
        this.element1 = e1;
        this.element2 = e2;
    }

    public object Element0 {
        get { return element0; }
    }

    public object Element1 {
        get { return element1; }
    }

    public object Element2 {
        get { return element2; }
    }

    public IEnumerator<object> GetEnumerator() {
        return DoIteration();
    }

    IEnumerator IEnumerable.GetEnumerator() {
        return DoIteration();
    }

    private IEnumerator<object> DoIteration() {
        for (int i = 0; i < Len; i++) {
            switch (i) {
                case 0:
                    yield return element0;
                    break;
                case 1:
                    yield return element1;
                    break;
                case 2:
                    yield return element2;
                    break;
            }
        }
    }
}
```

可以看出使用yield的版本明显简洁了很多。如果在迭代对象内部使用数组来存储element，则会更加简洁。

```
class IterableObjectWithYield : IEnumerable<object> {

    private object[] elements = new object[3];

    public const int Len = 3;

    public IterableObjectWithYield(object e0, object e1, object e2) {
        elements[0] = e0;
        elements[1] = e1;
        elements[2] = e2;
    }

    public object Element0 {
        get { return elements[0]; }
    }

    public object Element1 {
        get { return elements[1]; }
    }

    public object Element2 {
        get { return elements[2]; }
    }

    public IEnumerator<object> GetEnumerator() {
        return DoIteration();
    }

    IEnumerator IEnumerable.GetEnumerator() {
        return DoIteration();
    }

    private IEnumerator<object> DoIteration() {
        for (int i = 0; i < Len; i++) {
            yield return elements[i];
        }
    }
}
```

**实现迭代器只用了两行代码！**下面是简单的测试代码：

```
public class Client {

    public static void Main(string[] args) {
        Console.WriteLine("无yield");
        IterableObject obj0 = new IterableObject(0, 1, 2);
        foreach (object item in obj0) {
            Console.WriteLine(item);
        }

        Console.WriteLine("有yield");
        IterableObjectWithYield obj1 = new IterableObjectWithYield(3, 4, 5);
        foreach (object item in obj1) {
            Console.WriteLine(item);
        }
        
        Console.Read();
    }
}
```

C#中的yield大大降低了实现迭代器的代码成本，同时也使得实现协程更加方便，这在后文中会进一步叙述。

## Iterator & FSM(迭代器与有限状态机)

从上面的代码示例中，也可以清晰的看出，迭代器其实就是一个有限状态机，只不过其状态通常是迭代指针(`currentIndex`)的取值范围，而它的状态转换比较**单调简单**。所为单调是指其状态值之间的转换通常是线性单向的。

## Iterator & Continution(迭代器与协程)

协程是用户态的“线程”，其意义在于构建用户态的“轻量级线程”(相比OS内核线程和进程而言)，从而方便并发模式编程。

> 简言之，协程的全部精神就在于控制流的**主动**让出和恢复(yield/resume)。我们熟悉的子过程调用可以看作在返回时让出控制流的一种特殊的协程，其内部状态在返回时被丢弃了，因此不存在“恢复”这个操作。

协程的操作原语是`yield/resume`，此`yield`与迭代器的`yield`的操作语义实质上是相同的，均代表“出让控制权”。迭代器本身就是一种特殊的协程，其特殊之处在于它的状态恢复是“自动”的，并不需要**显示resume**。明白了两者的关系，对于Unity3D(C#)中的协程实现也就明了了。

关于协程更深入的讨论可以参看[这里][1]

[1]: http://blog.youxu.info/2014/12/04/coroutine/
