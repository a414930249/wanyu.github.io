mina是一款网络应用框架， 它是基于java nio 的 通讯框架， 提供抽象的事件驱动异步的api。 

mina 基本架构:

![image](http://hi.csdn.net/attachment/201203/14/0_1331696777kKre.gif)


大致上分为三层IoSerivce, Filter, Handler

首先我们来看一下Filter 

### 事件

前面说过， mina 是基于事件驱动的， 所以我们先来看一下mina定义了那些事件：  
IoEventType 类中定义了如下枚举变量：
```
public enum IoEventType {
    SESSION_CREATED,     session 创建
    SESSION_OPENED,      session 开启
    SESSION_CLOSED,      session 关闭
    MESSAGE_RECEIVED,    消息收到
    MESSAGE_SENT,        消息发送
    SESSION_IDLE,        session 闲置
    EXCEPTION_CAUGHT,    发生异常异常捕获
    WRITE,               写事件 out
    CLOSE,               关闭事件
}
```
这些事件对于用户使用者来说无需关心。
以上IoEventType仅是事件的类型（事件的属性）， mina对于事件是通过IoEvent对象进行封装的。  
来看一下IoEvent 都有哪些方法：
```java
public class IoEvent implements Runnable {
    private final IoEventType type;
    private final IoSession session;
    private final Object parameter;
    public IoEvent(IoEventType type, IoSession session, Object parameter)
    public IoEventType getType()
    public IoSession getSession()
    public Object getParameter()
    public void run() {
        fire();
    }
    public void fire()
```

我们看到IoEvent 实现了Runnable，也就是说这个对象会交给多线程去执行run方法。
在run方法中我们看到执行的是fire(), 也就是具体的任务了。 

下面来看一下fire中做的事情， 首先看以下IoEvent 中 具体做了哪些事情， 再看一下其子类IoFilterEvent又做了哪些事情。 后面我会讲IoFilterEvent 的作用是什么， 在哪里用到。


```
switch (getType()) {
    case MESSAGE_RECEIVED:
        getSession().getFilterChain().fireMessageReceived(getParameter());
        break;
    case MESSAGE_SENT:
        getSession().getFilterChain().fireMessageSent((WriteRequest) getParameter());
        break;
    case WRITE:
        getSession().getFilterChain().fireFilterWrite((WriteRequest) getParameter());
        break;
    case CLOSE:
        getSession().getFilterChain().fireFilterClose();
        break;
    .... 
}
```
其实做的事情就是触发（调用） session下filterchain的对应事件类型的方法。


##### IoFilterEvent  
IoFilterEvent 是IoEvent 的子类， 从名字就可以看出来， 其与IoFilter 有一定的关系， 所以IoFilterEvent是由IoFilter 产生的事件。
而IoFilter 产生的事件有什么特性呢？

在此插入IoFilter与IoFilterChain 的介绍。
###### IoFilter
IoFilter 是 拦截处理事件的过滤器， 如同servlet的filter。 它可以用于很多目的:  
日志记录， 性能测量， 认证等等。

###### IoFilterChain
而filterchain 即代表一串IoFilter的链子， 其有多个IoFilter 组成。 事实上，在IoFilterChain 中，封装了一层Entry, 每个Entry中封装了当前的IoFilter和下一个IoFilter的引用。  可以看到Entry的接口如下

```java
interface Entry {
    String getName();
    IoFilter getFilter();
    NextFilter getNextFilter();
    void addBefore(String name, IoFilter filter);
    void addAfter(String name, IoFilter filter);
    void replace(IoFilter newFilter);
}
```
    
###### IoFilter 和NextFilter 
NextFilter 顾名思义， 即下一个IoFilter,  为什么需要有这么一个接口呢？ 

事实上， 我们可以想象， nextFilter 是用来**触发** 下一个Iofilter的，其并不是IoFilter， 在IoFilter中， 我们可以看到这样的调用

```
void messageReceived(NextFilter nextFilter, IoSession session, Object message) throws Exception{
    ...
    nextFilter.messageReceived(session,message);
}

在DefaultIoFilterChain中我们可以看到Entry的实现里的nextFilter 是如下实现：
 this.nextFilter = new NextFilter() {
            ... 省去其他方法的实现
                public void messageReceived(IoSession session, Object message) {
                    Entry nextEntry = EntryImpl.this.nextEntry;
                    callNextMessageReceived(nextEntry, session, message);
                }

            };
        }
可以看到该方法是获取下一个Entry，我们知道下一个entry中包含下一个IoFilter，  
调用nextEntry的方法即调用entry中的Iofilter的方法：  
    private void callNextMessageReceived(Entry entry, IoSession session, Object message) {
        try {
            IoFilter filter = entry.getFilter();
            NextFilter nextFilter = entry.getNextFilter();
            filter.messageReceived(nextFilter, session, message);
        } catch (Exception e) {
            fireExceptionCaught(e);
        } catch (Error e) {
            fireExceptionCaught(e);
            throw e;
        }
    }
```

说完了IoFilter, IoFilterChain, 那与IoFilterEvent有什么关系呢？
因为是IoFilter产生的事件， 当触发这个事件时， 由于链式的特征， 该事件将触发当前IoFilter 以后的Filter, 所以我们看到， IoFilter fire()的实现其实就是调用下一个filter的对应的事件方法： 
```java
IoSession session = getSession();
NextFilter nextFilter = getNextFilter();
IoEventType type = getType();
switch (type) {
case MESSAGE_RECEIVED:
    Object parameter = getParameter();
    nextFilter.messageReceived(session, parameter);
... 
    
}
```

也可以这样理解， 一个事件的产生， 产生地就是头， 事件从当前头向链后传播。
IoEvent 做的是从chain 的开头往后传播， IoFilterEvent是由当前IoFilter产生往后传播。

#### 组装
说完了IoFilter， 来探讨一下如何将IoFilter组装成IoFilterChain:

其实里面的结构就是链表。
来看看chain中add的内容：
```
add 调用register
在这里说明一下， chain默认里面有2个entry， 1个是head头， 另一个是tail尾。
public synchronized void addLast(String name, IoFilter filter) {
    checkAddable(name);
    register(tail.prevEntry, name, filter);
}
所以addLast （从末尾添加) 有这么几步：
参数： 末尾的前一个entry， 名字， 当前filter
1. 创建一个entry， 该entry的前一个entry为末尾的前一个entry，后一个entry为尾entry,
2. 修改前一个entry的后一个entry的引用，指向第一步创建的entry
以上其实是链表基本的操作。

private void register(EntryImpl prevEntry, String name, IoFilter filter) {
    EntryImpl newEntry = new EntryImpl(prevEntry, prevEntry.nextEntry, name, filter);

    try {
        filter.onPreAdd(this, name, newEntry.getNextFilter());
    } catch (Exception e) {
        throw new IoFilterLifeCycleException("onPreAdd(): " + name + ':' + filter + " in " + getSession(), e);
    }

    prevEntry.nextEntry.prevEntry = newEntry;
    prevEntry.nextEntry = newEntry;
    name2entry.put(name, newEntry);

    try {
        filter.onPostAdd(this, name, newEntry.getNextFilter());
    } catch (Exception e) {
        deregister0(newEntry);
        throw new IoFilterLifeCycleException("onPostAdd(): " + name + ':' + filter + " in " + getSession(), e);
    }
}


```