---
title: 进程通信之AIDL
date: 2017-05-02 15:02:47
tags: Android
---
AIDL: Android Interface Definition Language 接口定义语言

## 进程间通信(IPC)方式
* startActivity传递Intent
* 文件
* Content Provider 
* Broadcast
* AIDL
* Messenger

### IPC支持数据格式
进程间通信支持的数据格式：

* java中基础数据类型(int，long，char...)
* String
* CharSequence
* List：ArrayList，List中元素必须是上述类型或者实现了parcelable接口的对象类型
* Map：HashMap，map中元素必须是上述类型或者实现了parcelable接口的对象类型
* 实现了parcelable接口的对象

## AIDL

#### 1.创建AIDL文件
*src/main/aidl/packagename/*下创建Book.aidl文件

```java
package com.example.android;

// Declare any non-default types here with import statements

parcelable Book;
```


*src/main/aidl/packagename/*下创建IBookService.aidl文件

```java
// IRemoteService.aidl
package com.example.android;

import com.example.android.Book;

// Declare any non-default types here with import statements

interface IBookService {
    List<Book> getBookList();
    void addBook(in Book book);
}
```
在Android Studio中，build之后会在*app/build/generated/source/aidl*目录下生成对应的IBookService.java文件
#### 2.实现接口
在java目录下生成一个Book.java的实例类，注意它必须要实现parcelable接口。

生成的IBookService.java实际上是一个接口类，它里面有一个叫Stub的实现类，继承至*android.os.Binder*.

Stub类中定义了一些有帮助的类，特别是*asInterface()*，它接收一个*IBinder*(通常是客户端的回调*onServiceConnected()*传过来的)参数，返回一个stub接口实例。

定义一个*BookService.java*的服务类，在它里面定义一个stub，*onBind()*方法中返回这个对象

```java
private List<Book> mBookList = new ArrayList<>();
private IBookService.Stub mBind = new IBookService.Stub() {
    @Override
    public List<Book> getBookList() throws RemoteException {
        return mBookList;
    }

    @Override
    public void addBook(Book book) throws RemoteException {
        if(!mBookList.contains(book)){
            mBookList.add(book);
        }
    }
};

@Override
public IBinder onBind(Intent intent) {
    return mBind;
}
```
#### 3.给客户端暴露接口
在上面的服务端中，已经通过*onBind()*方法已经返回了一个Stub。客户端调用*bindService()*来连接服务端，在*onServiceConnected()*的回调中会拿到这个Stub对象

```java
private IBookService mIBookService = null;
private ServiceConnection mServiceConnection = new ServiceConnection() {
    @Override
    public void onServiceConnected(ComponentName name, IBinder service) {
        mIBookService = IBookService.Stub.asInterface(service);

        try {
            List<Book> bookList = mIBookService.getBookList();
            LLog.d("Book size " + bookList.size());
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }

    @Override
    public void onServiceDisconnected(ComponentName name) {
    }
};
```
客户端拿到IBookService对象之后，就可以使用其中的方法了。