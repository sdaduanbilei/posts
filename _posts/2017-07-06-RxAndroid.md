### RxAndroid 记录

1.理解
```java
//第一步：创建观察者
Observer<String> observer = new Observer<String>() {
    @Override
    public void onSubscribe(Disposable d) {
    // 被观察者绑定到观察者的时的消息
    }
    @Override
    public void onNext(String s) {
    // 接收到被观察者发送的消息
    }
    @Override
    public void onError(Throwable e) {
   	// 接受发送消息过程中的错误信息
    }
    @Override
    public void onComplete() {
    // 接受被观察者已经发送完消息的消息
    }
};
//第二步：创建被观察者
Observable<String> observable = Observable.create(new ObservableOnSubscribe<String>() {
    @Override
    public void subscribe(ObservableEmitter<String> e) throws Exception {}
};
//第三部：绑定关系
observable.subscribe(observer);
// 调用just 方法
observable.just("Message");
//以上方法并未走到 onNext接受到消息的方法中


observable.just("Message").subscribe(observer)
// 接受到消息 内容

```
```Observable``` 简写
```java
Observable<String> observable = Observable.just("Message");
```
```onNext()``` 接收到消息


*  ```just(T...)``` 操作符 将传入的参数依次发送出来
```java
mObservable = Observable.just("1","2","3","4");
mObservable.subscribe(mObserver);
```

* ```from(T[])/from(Iterable<? extends T>)``` 将传入的数组或 Iterable 拆分成具体对象后，依次发送出来。


### Action
另外大部分接口方法都按照Java8的接口方法名进行了相应的修改，比如上面那个Consumer<T>接口原来叫Action1<T>，而Action2<T>改名成了BiConsumer

Action3-Action9被删掉了，大概因为没人用。。
其中Action类似于RxJava1.x中的Action0,区别在于Action允许抛出异常。

```java 
Consumer<String> consumer = new Consumer<String>() {
    @Override
    public void accept(String s) throws Exception {
        Log.d(TAG, "accept: "+s);
    }
};
Consumer<Throwable> consumer2 = new Consumer<Throwable>() {
    @Override
    public void accept(Throwable s) throws Exception {
        Log.d(TAG, "accept2: "+s);
    }
};
Action consumer3 = new Action() {
    @Override
    public void run() throws Exception {
        Log.d(TAG, "run: ");
    }
};
mObservable.just("1","2").subscribe(consumer,consumer2,consumer3);


// 用consumer 来替代 Observer 中的 onNext , consumer2 替代之前的 onError  用 consumer3 替代之前的onComplete
```


创建的Observer中多了一个回调方法onSubscribe，传递参数为Disposable ，Disposable相当于RxJava1.x中的Subscription,用于解除订阅。你可能纳闷为什么不像RxJava1.x中订阅时返回Disposable，而是选择回调出来呢。官方说是为了设计成Reactive-Streams架构。不过仔细想想这么一个场景还是很有用的，假设Observer需要在接收到异常数据项时解除订阅，在RxJava2.x中则非常简便，如下操作即可。
```java
Observer<Integer> observer = new Observer<Integer>() {
            private Disposable disposable;
 
            @Override
            public void onSubscribe(Disposable d) {
                disposable = d;
            }
 
            @Override
            public void onNext(Integer value) {
                Log.d("JG", value.toString());
                if (value > 3) {   // >3 时为异常数据，解除订阅
                    disposable.dispose();
                }
            }
 
            @Override
            public void onError(Throwable e) {
 
            }
 
            @Override
            public void onComplete() {
 
 
 
            }
        };
```
        
RxJava2.x中仍然保留了其他简化订阅方法，我们可以根据需求，选择相应的简化订阅。只不过传入的对象改为了Consumer。
```java  
Disposable disposable = observable.subscribe(new Consumer<Integer>() {
            @Override
            public void accept(Integer integer) throws Exception {
                  //这里接收数据项
            }
        }, new Consumer<Throwable>() {
            @Override
            public void accept(Throwable throwable) throws Exception {
              //这里接收onError
            }
        }, new Action() {
            @Override
            public void run() throws Exception {
              //这里接收onComplete。
            }
        });
```

### Flowable

Flowable是RxJava2.x中新增的类，专门用于应对背压（Backpressure）问题，但这并不是RxJava2.x中新引入的概念。所谓背压，即生产者的速度大于消费者的速度带来的问题，比如在Android中常见的点击事件，点击过快则会造成点击两次的效果。 
我们知道，在RxJava1.x中背压控制是由Observable完成的，使用如下：
```java
Observable.range(1,10000)
            .onBackpressureDrop()
            .subscribe(integer -> Log.d("JG",integer.toString()));
```

测试Flowable例子如下：
```java
Flowable.create(new FlowableOnSubscribe<Integer>() {
 
            @Override
            public void subscribe(FlowableEmitter<Integer> e) throws Exception {
 
                for(int i=0;i<10000;i++){
                    e.onNext(i);
                }
                e.onComplete();
            }
        }, FlowableEmitter.BackpressureMode.ERROR) //指定背压处理策略，抛出异常
                .subscribeOn(Schedulers.computation())
                .observeOn(Schedulers.newThread())
                .subscribe(new Consumer<Integer>() {
                    @Override
                    public void accept(Integer integer) throws Exception {
                        Log.d("JG", integer.toString());
                        Thread.sleep(1000);
                    }
                }, new Consumer<Throwable>() {
                    @Override
                    public void accept(Throwable throwable) throws Exception {
                        Log.d("JG",throwable.toString());
                    }
                });
```
或者可以使用类似RxJava1.x的方式来控制

```java
  Flowable.range(1,10000)
                .onBackpressureDrop()
                .subscribe(integer -> Log.d("JG",integer.toString()));
```
其中还需要注意的一点在于，Flowable并不是订阅就开始发送数据，而是需等到执行Subscription#request才能开始发送数据。当然，使用简化subscribe订阅方法会默认指定Long.MAX_VALUE。手动指定的例子如下：
```java
Flowable.range(1,10).subscribe(new Subscriber<Integer>() {
            @Override
            public void onSubscribe(Subscription s) {
                s.request(Long.MAX_VALUE);//设置请求数
            }
 
            @Override
            public void onNext(Integer integer) {
 
            }
 
            @Override
            public void onError(Throwable t) {
 
            }
 
            @Override
            public void onComplete() {
 
            }
        });
 ```
 
 ### Single
 不同于RxJava1.x中的SingleSubscriber,RxJava2中的SingleObserver多了一个回调方法onSubscribe。
 ```java
 interface SingleObserver<T> {
    void onSubscribe(Disposable d);
    void onSuccess(T value);
    void onError(Throwable error);
}
```

和Observable，Flowable一样会发送数据，不同的是订阅后只能接受到一次:

```java
Single<Long> single = Single.just(1l);

single.subscribe(new SingleObserver<Long>() {
    @Override
    public void onSubscribe(Disposable d) {
    }

    @Override
    public void onSuccess(Long value) {
        // 和onNext是一样的
    }

    @Override
    public void onError(Throwable e) {
    }
});
```
普通Observable可以使用toSingle转换:```Observable.just(1).toSingle()```


### Completable
同Single，Completable也被重新设计为Reactive-Streams架构，RxJava1.x的CompletableSubscriber改为CompletableObserver，源码如下
```java
interface CompletableObserver<T> {
    void onSubscribe(Disposable d);
    void onComplete();
    void onError(Throwable error);
}
```
同样也可以由普通的Observable转换而来:```Observable.just(1).toCompletable()```

### Subject/Processor
Processor和Subject的作用是相同的。关于Subject部分，RxJava1.x与RxJava2.x在用法上没有显著区别，这里就不介绍了。其中Processor是RxJava2.x新增的，继承自Flowable,所以支持背压控制。而Subject则不支持背压控制。使用如下：
```java
//Subject
        AsyncSubject<String> subject = AsyncSubject.create();
        subject.subscribe(o -> Log.d("JG",o));//three
        subject.onNext("one");
        subject.onNext("two");
        subject.onNext("three");
        subject.onComplete();
 
       //Processor
        AsyncProcessor<String> processor = AsyncProcessor.create();
        processor.subscribe(o -> Log.d("JG",o)); //three
        processor.onNext("one");
        processor.onNext("two");
        processor.onNext("three");
        processor.onComplete();
 ```
 
 RxJava1.x 如何平滑升级到RxJava2.x？ 
由于RxJava2.x变化较大无法直接升级，幸运的是，官方提供了RxJava2Interop这个库，可以方便地将RxJava1.x升级到RxJava2.x，或者将RxJava2.x转回RxJava1.x。地址：[https://github.com/akarnokd/RxJava2Interop](https://github.com/akarnokd/RxJava2Interop)

### 参考
* [http://www.jianshu.com/p/850af4f09b61](http://www.jianshu.com/p/850af4f09b61)
* [https://gank.io/post/560e15be2dca930e00da1083](https://gank.io/post/560e15be2dca930e00da1083)
* [http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2016/0907/6604.html](http://www.jcodecraeer.com/a/anzhuokaifa/androidkaifa/2016/0907/6604.html)





