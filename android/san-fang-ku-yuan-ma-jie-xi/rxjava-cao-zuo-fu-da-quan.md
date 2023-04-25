# RxJava操作符大全



> 本文基于RxJava 2.0文档，Github地址：[https://github.com/ReactiveX/RxJava](https://github.com/ReactiveX/RxJava)\
> 注意查看RxJava 1.x 与 2.x的不同

### 1 RxJava2的简单使用[¶](https://blog.yorek.xyz/android/other/RxJava/#1-rxjava2) <a href="#1-rxjava2" id="1-rxjava2"></a>

先上一段示例代码

```
Observable.create(new ObservableOnSubscribe<Integer>() {
    @Override
    public void subscribe(ObservableEmitter<Integer> emitter) throws Exception {
        Log.e(TAG, "subscribe thread = " + Thread.currentThread().getName());
        emitter.onNext(1);
        emitter.onNext(2);
        emitter.onNext(3);
        emitter.onNext(4);
        emitter.onComplete();
    }
})
        .observeOn(Schedulers.io())
        .map(new Function<Integer, String>() {
            @Override
            public String apply(Integer integer) throws Exception {
                Log.e(TAG, "apply thread = " + Thread.currentThread().getName() + ", integer = " + integer);
                return String.valueOf(integer);
            }
        })
        .subscribeOn(AndroidSchedulers.mainThread())
        .observeOn(Schedulers.newThread())
        .subscribe(new Observer<String>() {
            @Override
            public void onSubscribe(Disposable d) {
                Log.e(TAG, "onSubscribe thread = " + Thread.currentThread().getName());
            }

            @Override
            public void onNext(String s) {
                Log.e(TAG, "onNext thread = " + Thread.currentThread().getName() + ", String = " + s);
            }

            @Override
            public void onError(Throwable e) {
                Log.e(TAG, "onError thread = " + Thread.currentThread().getName() + ", Throwable = " + e);
            }

            @Override
            public void onComplete() {
                Log.e(TAG, "onComplete thread = " + Thread.currentThread().getName());
            }
        });
```

输出日志：

```
E/RxJavaActivity: onSubscribe thread = main
E/RxJavaActivity: subscribe thread = main
E/RxJavaActivity: apply thread = RxCachedThreadScheduler-1, integer = 1
E/RxJavaActivity: apply thread = RxCachedThreadScheduler-1, integer = 2
E/RxJavaActivity: apply thread = RxCachedThreadScheduler-1, integer = 3
E/RxJavaActivity: apply thread = RxCachedThreadScheduler-1, integer = 4
E/RxJavaActivity: onNext thread = RxNewThreadScheduler-1, String = 1
E/RxJavaActivity: onNext thread = RxNewThreadScheduler-1, String = 2
E/RxJavaActivity: onNext thread = RxNewThreadScheduler-1, String = 3
E/RxJavaActivity: onNext thread = RxNewThreadScheduler-1, String = 4
E/RxJavaActivity: onComplete thread = RxNewThreadScheduler-1
```

![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_schedulers.png)对应线程转化关系图

这就是RxJava使用的三部曲 1. 创建`Observable`\
创建`Observable`时，回调的是`ObservableEmitter`，即发射器，用于发射数据(`onNext`)和通知(`onError/onComplete`) 2. 创建`Observer`\
创建的`Observer`中有一个回调方法`onSubscribe`，传递参数为`Disposable`，可用于解除订阅。 3. 建立订阅关系`observable.subscribe(observer)`

RxJava2中仍然保留了其他简化订阅方法，我们可以根据需求，选择相应的简化订阅(`Consumer`)。

同时，RxJava2引入了新的类`Flowable`，专门用于应对背压(backpressure)问题，但这并不是RxJava2.x中新引入的概念。所谓背压，即生产者的速度大于消费者的速度带来的问题，比如在Android中常见的点击事件，点击过快则会造成点击两次的效果。\
在RxJava2.x中将其独立了出来，取名为`Flowable`。因此，`Observable`已经不具备背压处理能力。

> 关于backpressure，官方文档地址 [1](https://github.com/ReactiveX/RxJava/wiki/Backpressure) [2](https://github.com/ReactiveX/RxJava/wiki/Backpressure-\(2.0\))

### 2 RxJava中的操作符[¶](https://blog.yorek.xyz/android/other/RxJava/#2-rxjava) <a href="#2-rxjava" id="2-rxjava"></a>

#### 2.1 [阻塞操作](https://github.com/ReactiveX/RxJava/wiki/Blocking-Observable-Operators)[¶](https://blog.yorek.xyz/android/other/RxJava/#21) <a href="#21" id="21"></a>

当阻塞的`Observables`执行完成后，其他代码才能执行.

* [forEach](http://reactivex.io/documentation/operators/subscribe.html) - invoke a function on each item emitted by the Observable; block until the Observable completes
* [first/firstOrDefault](http://reactivex.io/documentation/operators/first.html) - block until the Observable emits an item, then return the first item emitted by the Observable or a default item if the Observable did not emit an item![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_first.png)
* [last/lastOrDefault](http://reactivex.io/documentation/operators/last.html) - block until the Observable completes, then return the last item emitted by the Observable or a default item if there is no last item![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_last.png)
* [mostRecent](http://reactivex.io/documentation/operators/first.html) - returns an iterable that always returns the item most recently emitted by the Observable
* [next](http://reactivex.io/documentation/operators/takelast.html) - returns an iterable that blocks until the Observable emits another item, then returns that item
* [latest](http://reactivex.io/documentation/operators/first.html) - returns an iterable that blocks until or unless the Observable emits an item that has not been returned by the iterable, then returns that item
* [single](http://reactivex.io/documentation/operators/first.html) - if the Observable completes after emitting a single item, return that item, otherwise throw an exception
* [singleOrDefault](http://reactivex.io/documentation/operators/first.html) - if the Observable completes after emitting a single item, return that item, otherwise return a default item
* [toFuture](http://reactivex.io/documentation/operators/to.html) - convert the Observable into a Future![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_to.png)
* [toIterable](http://reactivex.io/documentation/operators/to.html) - convert the sequence emitted by the Observable into an Iterable
* [getIterator](http://reactivex.io/documentation/operators/to.html) - convert the sequence emitted by the Observable into an Iterator

示例代码

```
Observable.just(1, 2, 3).observeOn(Schedulers.io()).blockingForEach(new Consumer<Integer>() {
    @Override
    public void accept(Integer integer) throws Exception {
        Log.e(TAG, integer + " - " + Thread.currentThread().getName());
    }
});
Log.e(TAG, "done - " + Thread.currentThread().getName());
```

结果\


```
02-12 16:33:13.083 7539-7539/? E/RxJavaActivity: 1 - main
02-12 16:33:13.083 7539-7539/? E/RxJavaActivity: 2 - main
02-12 16:33:13.083 7539-7539/? E/RxJavaActivity: 3 - main
02-12 16:33:13.087 7539-7539/? E/RxJavaActivity: done - main
```

> **注意**: RxJava2中针对此部分有了变化：\
> _toBlocking().y - inlined as blockingY() operators, except toFuture_\
> 也就是说，在RxJava2中使用上述操作符，应该是这样的`Observable.just(...).blockingForEach`。即使用时加上前缀`blockingXXX`

#### 2.2 [组合操作](https://github.com/ReactiveX/RxJava/wiki/Combining-Observables)[¶](https://blog.yorek.xyz/android/other/RxJava/#22) <a href="#22" id="22"></a>

* [startWith](http://reactivex.io/documentation/operators/startwith.html) - emit a specified sequence of items before beginning to emit the items from the Observable![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_startwith.png)
* [merge](http://reactivex.io/documentation/operators/merge.html) - combine multiple Observables into one![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_merge.png)
* [mergeDelayError](http://reactivex.io/documentation/operators/merge.html) - combine multiple Observables into one, allowing error-free Observables to continue before propagating errors![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_merge\_delay\_error.png)
* [zip](http://reactivex.io/documentation/operators/zip.html) - combine sets of items emitted by two or more Observables together via a specified function and emit items based on the results of this function![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_zip.png)
* [combineLatest](http://reactivex.io/documentation/operators/combinelatest.html) - when an item is emitted by either of two Observables, combine the latest item emitted by each Observable via a specified function and emit items based on the results of this function![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_combine\_lastest.png)
* [join and groupJoin](http://reactivex.io/documentation/operators/join.html) - combine the items emitted by two Observables whenever one item from one Observable falls within a window of duration specified by an item emitted by the other Observable\
  如果一个Observable发射了一条数据，只要在另一个Observable发射的数据定义的时间窗口内，就结合两个Observable发射的数据，然后发射结合后的数据。\
  目标Observable和源Observable发射的数据都有一个有效时间限制，比如目标发射了一条数据（a）有效期为3s，过了2s后，源发射了一条数据（b），因为2s<3s，目标的那条数据还在有效期，所以可以组合为ab；再过2s，源又发射了一条数据（c）,这时候一共过去了4s，目标的数据a已经过期，所以不能组合了…\
  ![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_join.png)![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_group\_join.png)
* [switchOnNext](http://reactivex.io/documentation/operators/switch.html) - convert an Observable that emits Observables into a single Observable that emits the items emitted by the most-recently emitted of those Observables\
  将一个发射多个Observables的Observable转换成另一个单独的Observable，后者发射那些Observables最近发射的数据项。\
  Switch订阅一个发射多个Observables的Observable。它每次观察那些Observables中的一个，Switch返回的这个Observable取消订阅前一个发射数据的Observable，开始发射最近的Observable发射的数据。\
  注意：当原始Observable发射了一个新的Observable时（不是这个新的Observable发射了一条数据时），它将取消订阅之前的那个Observable。这意味着，在后来那个Observable产生之后，前一个Observable发射的数据将被丢弃（就像图例上的那个黄色圆圈一样）。\
  ![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_switch.png)

组合操作例子：

```
Observable.just(1, 2, 3)
          .startWith(0)
          .subscribe(new Consumer<Integer>() {
              @Override
              public void accept(Integer integer) throws Exception {
                  Log.e(TAG, "accept " + integer);
              }
          });
```

#### 2.3 [条件与Boolean操作符](https://github.com/ReactiveX/RxJava/wiki/Conditional-and-Boolean-Operators)[¶](https://blog.yorek.xyz/android/other/RxJava/#23-boolean) <a href="#23-boolean" id="23-boolean"></a>

条件操作符 - [amb](http://reactivex.io/documentation/operators/amb.html) — given two or more source Observables, emits all of the items from the first of these Observables to emit an item![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_amb.png)- [defaultIfEmpty](http://reactivex.io/documentation/operators/defaultifempty.html) — emit items from the source Observable, or emit a default item if the source Observable completes after emitting no items![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_default\_if\_empty.png)- [skipUntil](http://reactivex.io/documentation/operators/skipuntil.html) — discard items emitted by a source Observable until a second Observable emits an item, then emit the remainder of the source Observable's items![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_skip\_until.png)- [skipWhile](http://reactivex.io/documentation/operators/skipwhile.html) — discard items emitted by an Observable until a specified condition is false, then emit the remainder![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_skip\_while.png)- [takeUntil](http://reactivex.io/documentation/operators/takeuntil.html) — emits the items from the source Observable until a second Observable emits an item or issues a notification![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_take\_until.png)- [takeWhile and takeWhileWithIndex](http://reactivex.io/documentation/operators/takewhile.html) — emit items emitted by an Observable as long as a specified condition is true, then skip the remainder![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_take\_while.png)

Boolean操作符 - [all](http://reactivex.io/documentation/operators/all.html) — determine whether all items emitted by an Observable meet some criteria![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_all.png)- [contains](http://reactivex.io/documentation/operators/contains.html) — determine whether an Observable emits a particular item or not![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_contain.png)- [exists and isEmpty](http://reactivex.io/documentation/operators/contains.html) — determine whether an Observable emits any items or not - [sequenceEqual](http://reactivex.io/documentation/operators/sequenceequal.html) — test the equality of the sequences emitted by two Observables![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_sequence\_equal.png)

#### 2.4 [Connectable Observable](https://github.com/ReactiveX/RxJava/wiki/Connectable-Observable-Operators)[¶](https://blog.yorek.xyz/android/other/RxJava/#24-connectable-observable) <a href="#24-connectable-observable" id="24-connectable-observable"></a>

A Connectable Observable resembles an ordinary Observable, except that it does not begin emitting items when it is subscribed to, but only when its `connect()` method is called. In this way you can wait for all intended Subscribers to subscribe to the Observable before the Observable begins emitting items.![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_publish.png)

* [ConnectableObservable.connect](http://reactivex.io/documentation/operators/connect.html) — instructs a Connectable Observable to begin emitting items
* [Observable.publish](http://reactivex.io/documentation/operators/publish.html) — represents an Observable as a Connectable Observable
* [Observable.replay](http://reactivex.io/documentation/operators/replay.html) — ensures that all Subscribers see the same sequence of emitted items, even if they subscribe after the Observable begins emitting the items![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_replay.png)
* [ConnectableObservable.refCount](http://reactivex.io/documentation/operators/refcount.html) — makes a Connectable Observable behave like an ordinary Observable![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_ref\_count.png)

#### 2.5 [Error Handling Operators](https://github.com/ReactiveX/RxJava/wiki/Error-Handling-Operators)[¶](https://blog.yorek.xyz/android/other/RxJava/#25-error-handling-operators) <a href="#25-error-handling-operators" id="25-error-handling-operators"></a>

There are a variety of operators that you can use to react to or recover from onError notifications from Observables. For example, you might: 1. swallow the error and switch over to a backup Observable to continue the sequence 2. swallow the error and emit a default item 3. swallow the error and immediately try to restart the failed Observable 4. swallow the error and try to restart the failed Observable after some back-off interval

The following pages explain these operators. - [onErrorResumeNext](http://reactivex.io/documentation/operators/catch.html) — instructs an Observable to emit a sequence of items if it encounters an error![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_catch.png)- [onErrorReturn](http://reactivex.io/documentation/operators/catch.html) — instructs an Observable to emit a particular item when it encounters an error - [onExceptionResumeNext](http://reactivex.io/documentation/operators/catch.html) — instructs an Observable to continue emitting items after it encounters an exception (but not another variety of throwable) - [retry](http://reactivex.io/documentation/operators/retry.html) — if a source Observable emits an error, resubscribe to it in the hopes that it will complete without error![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_retry.png)- [retryWhen](http://reactivex.io/documentation/operators/retry.html) — if a source Observable emits an error, pass that error to another Observable to determine whether to resubscribe to the source

#### 2.6 [Filtering](https://github.com/ReactiveX/RxJava/wiki/Filtering-Observables)[¶](https://blog.yorek.xyz/android/other/RxJava/#26-filtering) <a href="#26-filtering" id="26-filtering"></a>

* [filter](http://reactivex.io/documentation/operators/filter.html) — filter items emitted by an Observable![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_filter.png)
* [takeLast](http://reactivex.io/documentation/operators/takelast.html) — only emit the last n items emitted by an Observable![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_take\_last.png)
* [last](http://reactivex.io/documentation/operators/last.html) — emit only the last item emitted by an Observable![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_last.png)
* [lastOrDefault](http://reactivex.io/documentation/operators/last.html) — emit only the last item emitted by an Observable, or a default value if the source Observable is empty
* [takeLastBuffer](http://reactivex.io/documentation/operators/takelast.html) — emit the last n items emitted by an Observable, as a single list item
* [skip](http://reactivex.io/documentation/operators/skip.html) — ignore the first n items emitted by an Observable![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_skip.png)
* [skipLast](http://reactivex.io/documentation/operators/skiplast.html) — ignore the last n items emitted by an Observable![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_skip\_last.png)
* [take](http://reactivex.io/documentation/operators/take.html) — emit only the first n items emitted by an Observable![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_take.png)
* [first and takeFirst](http://reactivex.io/documentation/operators/first.html) — emit only the first item emitted by an Observable, or the first item that meets some condition![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_first.png)
* [firstOrDefault](http://reactivex.io/documentation/operators/first.html) — emit only the first item emitted by an Observable, or the first item that meets some condition, or a default value if the source Observable is empty
* [elementAt](http://reactivex.io/documentation/operators/elementat.html) — emit item n emitted by the source Observable![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_element\_at.png)
* [elementAtOrDefault](http://reactivex.io/documentation/operators/elementat.html) — emit item n emitted by the source Observable, or a default item if the source Observable emits fewer than n items
* [sample or throttleLast](http://reactivex.io/documentation/operators/sample.html) — emit the most recent items emitted by an Observable within periodic time intervals![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_sample.png)
* [throttleFirst](http://reactivex.io/documentation/operators/sample.html) — emit the first items emitted by an Observable within periodic time intervals
* [throttleWithTimeout or debounce](http://reactivex.io/documentation/operators/debounce.html) — only emit an item from the source Observable after a particular timespan has passed without the Observable emitting any other items![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_debounce.png)
* [timeout](http://reactivex.io/documentation/operators/timeout.html) — emit items from a source Observable, but issue an exception if no item is emitted in a specified timespan![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_timeout.png)
* [distinct](http://reactivex.io/documentation/operators/distinct.html) — suppress duplicate items emitted by the source Observable![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_distinct.png)
* [distinctUntilChanged](http://reactivex.io/documentation/operators/distinct.html) — suppress duplicate consecutive items emitted by the source Observable
* [ofType](http://reactivex.io/documentation/operators/filter.html) — emit only those items from the source Observable that are of a particular class
* [ignoreElements](http://reactivex.io/documentation/operators/ignoreelements.html) — discard the items emitted by the source Observable and only pass through the error or completed notification![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_ignore\_elements.png)

#### 2.7 [聚合操作Aggregate](https://github.com/ReactiveX/RxJava/wiki/Mathematical-and-Aggregate-Operators)[¶](https://blog.yorek.xyz/android/other/RxJava/#27-aggregate) <a href="#27-aggregate" id="27-aggregate"></a>

* [concat](http://reactivex.io/documentation/operators/concat.html) — concatenate two or more Observables sequentially![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_concat.png)
* [count and countLong](http://reactivex.io/documentation/operators/count.html) — counts the number of items emitted by an Observable and emits this count![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_count.png)
* [reduce](http://reactivex.io/documentation/operators/reduce.html) — apply a function to each emitted item, sequentially, and emit only the final accumulated value![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_reduce.png)
* [collect](http://reactivex.io/documentation/operators/reduce.html) — collect items emitted by the source Observable into a single mutable data structure and return an Observable that emits this structure
* [toList](http://reactivex.io/documentation/operators/to.html) — collect all items from an Observable and emit them as a single List![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_to.png)
* [toSortedList](http://reactivex.io/documentation/operators/to.html) — collect all items from an Observable and emit them as a single, sorted List
* [toMap](http://reactivex.io/documentation/operators/to.html) — convert the sequence of items emitted by an Observable into a map keyed by a specified key function
* [toMultiMap](http://reactivex.io/documentation/operators/to.html) — convert the sequence of items emitted by an Observable into an ArrayList that is also a map keyed by a specified key function

#### 2.8 [Observable Creation](https://github.com/ReactiveX/RxJava/wiki/Creating-Observables)[¶](https://blog.yorek.xyz/android/other/RxJava/#28-observable-creation) <a href="#28-observable-creation" id="28-observable-creation"></a>

* [just](http://reactivex.io/documentation/operators/just.html) — convert an object or several objects into an Observable that emits that object or those objects![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_just.png)
* [from](http://reactivex.io/documentation/operators/from.html) — convert an Iterable, a Future, or an Array into an Observable![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_from.png)
* [create](http://reactivex.io/documentation/operators/create.html) — advanced use only! create an Observable from scratch by means of a function, consider fromEmitter instead![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_create.png)
* [fromEmitter](http://reactivex.io/RxJava/javadoc/rx/Observable.html#fromEmitter\(rx.functions.Action1,%20rx.AsyncEmitter.BackpressureMode\)) — create safe, backpressure-enabled, unsubscription-supporting Observable via a function and push events.
* [defer](http://reactivex.io/documentation/operators/defer.html) — do not create the Observable until a Subscriber subscribes; create a fresh Observable on each subscription![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_defer.png)
* [range](http://reactivex.io/documentation/operators/range.html) — create an Observable that emits a range of sequential integers![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_range.png)
* [interval](http://reactivex.io/documentation/operators/interval.html) — create an Observable that emits a sequence of integers spaced by a given time interval![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_interval.png)
* [timer](http://reactivex.io/documentation/operators/timer.html) — create an Observable that emits a single item after a given delay![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_timer.png)
* [empty](http://reactivex.io/documentation/operators/empty-never-throw.html) — create an Observable that emits nothing and then completes![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_empty.png)
* [error](http://reactivex.io/documentation/operators/empty-never-throw.html) — create an Observable that emits nothing and then signals an error![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_throw.png)
* [never](http://reactivex.io/documentation/operators/empty-never-throw.html) — create an Observable that emits nothing at all![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_never.png)

#### 2.9 [Transformational](https://github.com/ReactiveX/RxJava/wiki/Transforming-Observables)[¶](https://blog.yorek.xyz/android/other/RxJava/#29-transformational) <a href="#29-transformational" id="29-transformational"></a>

* [map](http://reactivex.io/documentation/operators/map.html) — transform the items emitted by an Observable by applying a function to each of them![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_map.png)
* [flatMap, concatMap, and flatMapIterable](http://reactivex.io/documentation/operators/flatmap.html) — transform the items emitted by an Observable into Observables (or Iterables), then flatten this into a single Observable![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_flat\_map.png)
* [switchMap](http://reactivex.io/documentation/operators/flatmap.html) — transform the items emitted by an Observable into Observables, and mirror those items emitted by the most-recently transformed Observable
* [scan](http://reactivex.io/documentation/operators/scan.html) — apply a function to each item emitted by an Observable, sequentially, and emit each successive value![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_scan.png)
* [groupBy](http://reactivex.io/documentation/operators/groupby.html) — divide an Observable into a set of Observables that emit groups of items from the original Observable, organized by key![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_group\_by.png)
* [buffer](http://reactivex.io/documentation/operators/buffer.html) — periodically gather items from an Observable into bundles and emit these bundles rather than emitting the items one at a time![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_buffer.png)
* [window](http://reactivex.io/documentation/operators/window.html) — periodically subdivide items from an Observable into Observable windows and emit these windows rather than emitting the items one at a time![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_window.png)
* [cast](http://reactivex.io/documentation/operators/map.html) — cast all items from the source Observable into a particular type before reemitting them

#### 2.10 [Utility Operators](https://github.com/ReactiveX/RxJava/wiki/Observable-Utility-Operators)[¶](https://blog.yorek.xyz/android/other/RxJava/#210-utility-operators) <a href="#210-utility-operators" id="210-utility-operators"></a>

* [materialize](http://reactivex.io/documentation/operators/materialize-dematerialize.html) — convert an Observable into a list of Notifications![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_materialize.png)
* [dematerialize](http://reactivex.io/documentation/operators/materialize-dematerialize.html) — convert a materialized Observable back into its non-materialized form![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_dematerialize.png)
* [timestamp](http://reactivex.io/documentation/operators/timestamp.html) — attach a timestamp to every item emitted by an Observable![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_timestamp.png)
* [serialize](http://reactivex.io/documentation/operators/serialize.html) — force an Observable to make serialized calls and to be well-behaved![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_serialize.png)
* [cache](http://reactivex.io/documentation/operators/replay.html) — remember the sequence of items emitted by the Observable and emit the same sequence to future Subscribers
* [observeOn](http://reactivex.io/documentation/operators/observeon.html) — specify on which Scheduler a Subscriber should observe the Observable![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_schedulers.png)
* [subscribeOn](http://reactivex.io/documentation/operators/subscribeon.html) — specify which Scheduler an Observable should use when its subscription is invoked
* [doOnEach](http://reactivex.io/documentation/operators/do.html) — register an action to take whenever an Observable emits an item![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_do.png)
* [doOnCompleted](http://reactivex.io/documentation/operators/do.html) — register an action to take when an Observable completes successfully
* [doOnError](http://reactivex.io/documentation/operators/do.html) — register an action to take when an Observable completes with an error
* [doOnTerminate](http://reactivex.io/documentation/operators/do.html) — register an action to take when an Observable completes, either successfully or with an error
* [doOnSubscribe](http://reactivex.io/documentation/operators/do.html) — register an action to take when an observer subscribes to an Observable
* [doOnUnsubscribe](http://reactivex.io/documentation/operators/do.html) — register an action to take when an observer unsubscribes from an Observable
* [finallyDo](http://reactivex.io/documentation/operators/do.html) — register an action to take when an Observable completes
* [delay](http://reactivex.io/documentation/operators/delay.html) — shift the emissions from an Observable forward in time by a specified amount![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_delay.png)
* [delaySubscription](http://reactivex.io/documentation/operators/delay.html) — hold an Subscriber's subscription request for a specified amount of time before passing it on to the source Observable
* [timeInterval](http://reactivex.io/documentation/operators/timeinterval.html) — emit the time lapsed between consecutive emissions of a source Observable![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_time\_interval.png)
* [using](http://reactivex.io/documentation/operators/using.html) — create a disposable resource that has the same lifespan as an Observable
* [single](http://reactivex.io/documentation/operators/first.html) — if the Observable completes after emitting a single item, return that item, otherwise throw an exception
* [singleOrDefault](http://reactivex.io/documentation/operators/first.html) — if the Observable completes after emitting a single item, return that item, otherwise return a default item
* [repeat](http://reactivex.io/documentation/operators/repeat.html) — create an Observable that emits a particular item or sequence of items repeatedly![](https://blog.yorek.xyz/assets/images/android/rxjava/rxjava\_repeat.png)
* [repeatWhen](http://reactivex.io/documentation/operators/repeat.html) — create an Observable that emits a particular item or sequence of items repeatedly, depending on the emissions of a second Observable

### 3 [操作符决策树](http://reactivex.io/documentation/operators.html#alphabetical)[¶](https://blog.yorek.xyz/android/other/RxJava/#3) <a href="#3" id="3"></a>

This tree can help you find the ReactiveX Observable operator you’re looking for.

**I want to create a new Observable**\
 …that emits a particular item: [Just](http://reactivex.io/documentation/operators/just.html)\
  …that was returned from a function called at subscribe-time: [Start](http://reactivex.io/documentation/operators/start.html)\
  …that was returned from an `Action`, `Callable`, `Runnable`, or something of that sort, called at subscribe-time: [From](http://reactivex.io/documentation/operators/from.html)\
  …after a specified delay: [Timer](http://reactivex.io/documentation/operators/timer.html)\
 …that pulls its emissions from a particular `Array`, `Iterable`, or something like that: [From](http://reactivex.io/documentation/operators/from.html)\
 …by retrieving it from a Future: [Start](http://reactivex.io/documentation/operators/start.html)\
 …that obtains its sequence from a Future: [From](http://reactivex.io/documentation/operators/from.html)\
 …that emits a sequence of items repeatedly: [Repeat](http://reactivex.io/documentation/operators/repeat.html)\
 …from scratch, with custom logic: [Create](http://reactivex.io/documentation/operators/create.html)\
 …for each observer that subscribes: [Defer](http://reactivex.io/documentation/operators/defer.html)\
 …that emits a sequence of integers: [Range](http://reactivex.io/documentation/operators/range.html)\
  …at particular intervals of time: [Interval](http://reactivex.io/documentation/operators/interval.html)\
   …after a specified delay: [Timer](http://reactivex.io/documentation/operators/timer.html)\
 …that completes without emitting items: [Empty](http://reactivex.io/documentation/operators/empty-never-throw.html)\
 …that does nothing at all: [Never](http://reactivex.io/documentation/operators/empty-never-throw.html)

**I want to create an Observable by combining other Observables**\
 …and emitting all of the items from all of the Observables in whatever order they are received: [Merge](http://reactivex.io/documentation/operators/merge.html)\
 …and emitting all of the items from all of the Observables, one Observable at a time: [Concat](http://reactivex.io/documentation/operators/concat.html)\
 …by combining the items from two or more Observables sequentially to come up with new items to emit\
  …whenever each of the Observables has emitted a new item: [Zip](http://reactivex.io/documentation/operators/zip.html)\
  …whenever any of the Observables has emitted a new item: [CombineLatest](http://reactivex.io/documentation/operators/combinelatest.html)\
  …whenever an item is emitted by one Observable in a window defined by an item emitted by another: [Join](http://reactivex.io/documentation/operators/join.html)\
  …by means of `Pattern` and `Plan` intermediaries: [And/Then/When](http://reactivex.io/documentation/operators/and-then-when.html)\
 …and emitting the items from only the most-recently emitted of those Observables: [Switch](http://reactivex.io/documentation/operators/switch.html)

**I want to emit the items from an Observable after transforming them**\
 …one at a time with a function: [Map](http://reactivex.io/documentation/operators/map.html)\
 …by emitting all of the items emitted by corresponding Observables: [FlatMap](http://reactivex.io/documentation/operators/flatmap.html)\
  …one Observable at a time, in the order they are emitted: [ConcatMap](http://reactivex.io/documentation/operators/flatmap.html)\
 …based on all of the items that preceded them: [Scan](http://reactivex.io/documentation/operators/scan.html)\
 …by attaching a timestamp to them: [Timestamp](http://reactivex.io/documentation/operators/timestamp.html)\
 …into an indicator of the amount of time that lapsed before the emission of the item: [TimeInterval](http://reactivex.io/documentation/operators/timeinterval.html)

**I want to shift the items emitted by an Observable forward in time before reemitting them**: [Delay](http://reactivex.io/documentation/operators/delay.html)

**I want to transform items and notifications from an Observable into items and reemit them**\
 …by wrapping them in `Notification` objects: [Materialize](http://reactivex.io/documentation/operators/materialize-dematerialize.html)\
  …which I can then unwrap again with: [Dematerialize](http://reactivex.io/documentation/operators/materialize-dematerialize.html)

**I want to ignore all items emitted by an Observable and only pass along its completed/error notification**: [IgnoreElements](http://reactivex.io/documentation/operators/ignoreelements.html)

**I want to mirror an Observable but prefix items to its sequence**: [StartWith](http://reactivex.io/documentation/operators/startwith.html)\
 …only if its sequence is empty: [DefaultIfEmpty](http://reactivex.io/documentation/operators/defaultifempty.html)

**I want to collect items from an Observable and reemit them as buffers of items**: [Buffer](http://reactivex.io/documentation/operators/buffer.html)\
 …containing only the last items emitted: [TakeLastBuffer](http://reactivex.io/documentation/operators/takelast.html)

**I want to split one Observable into multiple Observables**: [Window](http://reactivex.io/documentation/operators/window.html)\
 …so that similar items end up on the same Observable: [GroupBy](http://reactivex.io/documentation/operators/groupby.html)

**I want to retrieve a particular item emitted by an Observable:**\
 …the last item emitted before it completed: [Last](http://reactivex.io/documentation/operators/last.html)\
 …the sole item it emitted: [Single](http://reactivex.io/documentation/operators/first.html)\
 …the first item it emitted: [First](http://reactivex.io/documentation/operators/first.html)

**I want to reemit only certain items from an Observable**\
 …by filtering out those that do not match some predicate: [Filter](http://reactivex.io/documentation/operators/filter.html)\
 …that is, only the first item: [First](http://reactivex.io/documentation/operators/first.html)\
 …that is, only the first item\*s\*: [Take](http://reactivex.io/documentation/operators/take.html)\
 …that is, only the last item: [Last](http://reactivex.io/documentation/operators/last.html)\
 …that is, only item _n_: [ElementAt](http://reactivex.io/documentation/operators/elementat.html)\
 …that is, only those items after the first items\
  …that is, after the first _n_ items: [Skip](http://reactivex.io/documentation/operators/skip.html)\
  …that is, until one of those items matches a predicate: [SkipWhile](http://reactivex.io/documentation/operators/skipwhile.html)\
  …that is, after an initial period of time: [Skip](http://reactivex.io/documentation/operators/skip.html)\
  …that is, after a second Observable emits an item: [SkipUntil](http://reactivex.io/documentation/operators/skipuntil.html)\
 …that is, those items except the last items\
  …that is, except the last _n_ items: [SkipLast](http://reactivex.io/documentation/operators/skiplast.html)\
  …that is, until one of those items matches a predicate: [TakeWhile](http://reactivex.io/documentation/operators/takewhile.html)\
  …that is, except items emitted during a period of time before the source completes: [SkipLast](http://reactivex.io/documentation/operators/skiplast.html)\
  …that is, except items emitted after a second Observable emits an item: [TakeUntil](http://reactivex.io/documentation/operators/takeuntil.html)\
 …by sampling the Observable periodically: [Sample](http://reactivex.io/documentation/operators/sample.html)\
 …by only emitting items that are not followed by other items within some duration: [Debounce](http://reactivex.io/documentation/operators/debounce.html)\
 …by suppressing items that are duplicates of already-emitted items: [Distinct](http://reactivex.io/documentation/operators/distinct.html)\
  …if they immediately follow the item they are duplicates of: [DistinctUntilChanged](http://reactivex.io/documentation/operators/distinct.html)\
 …by delaying my subscription to it for some time after it begins emitting items: [DelaySubscription](http://reactivex.io/documentation/operators/delay.html)

**I want to reemit items from an Observable only on condition that it was the first of a collection of Observables to emit an item**: [Amb](http://reactivex.io/documentation/operators/amb.html)

**I want to evaluate the entire sequence of items emitted by an Observable**\
 …and emit a single boolean indicating if _all_ of the items pass some test: [All](http://reactivex.io/documentation/operators/all.html)\
 …and emit a single boolean indicating if the Observable emitted _any_ item (that passes some test): [Contains](http://reactivex.io/documentation/operators/contains.html)\
 …and emit a single boolean indicating if the Observable emitted _no_ items: [IsEmpty](http://reactivex.io/documentation/operators/contains.html)\
 …and emit a single boolean indicating if the sequence is identical to one emitted by a second Observable: [SequenceEqual](http://reactivex.io/documentation/operators/sequenceequal.html)\
 …and emit the average of all of their values: [Average](http://reactivex.io/documentation/operators/average.html)\
 …and emit the sum of all of their values: [Sum](http://reactivex.io/documentation/operators/sum.html)\
 …and emit a number indicating how many items were in the sequence: [Count](http://reactivex.io/documentation/operators/count.html)\
 …and emit the item with the maximum value: [Max](http://reactivex.io/documentation/operators/max.html)\
 …and emit the item with the minimum value: [Min](http://reactivex.io/documentation/operators/min.html)\
 …by applying an aggregation function to each item in turn and emitting the result: [Scan](http://reactivex.io/documentation/operators/scan.html)

**I want to convert the entire sequence of items emitted by an Observable into some other data structure**: [To](http://reactivex.io/documentation/operators/to.html)

**I want an operator to operate on a particular** [Scheduler](http://reactivex.io/scheduler.html): [SubscribeOn](http://reactivex.io/documentation/operators/subscribeon.html)\
 …when it notifies observers: [ObserveOn](http://reactivex.io/documentation/operators/observeon.html)

**I want an Observable to invoke a particular action when certain events occur**: [Do](http://reactivex.io/documentation/operators/do.html)

**I want an Observable that will notify observers of an error**: [Throw](http://reactivex.io/documentation/operators/empty-never-throw.html)\
 …if a specified period of time elapses without it emitting an item: [Timeout](http://reactivex.io/documentation/operators/timeout.html)

**I want an Observable to recover gracefully**\
 …from a timeout by switching to a backup Observable: [Timeout](http://reactivex.io/documentation/operators/timeout.html)\
 …from an upstream error notification: [Catch](http://reactivex.io/documentation/operators/catch.html)\
  …by attempting to resubscribe to the upstream Observable: [Retry](http://reactivex.io/documentation/operators/retry.html)

**I want to create a resource that has the same lifespan as the Observable**: [Using](http://reactivex.io/documentation/operators/using.html)

**I want to subscribe to an Observable and receive a `Future` that blocks until the Observable completesStartI want an Observable that does not start emitting items to subscribers until asked**: [Publish](http://reactivex.io/documentation/operators/publish.html)\
 …and then only emits the last item in its sequence: [PublishLast](http://reactivex.io/documentation/operators/publish.html)\
 …and then emits the complete sequence, even to those who subscribe after the sequence has begun: [Replay](http://reactivex.io/documentation/operators/replay.html)\
 …but I want it to go away once all of its subscribers unsubscribe: [RefCount](http://reactivex.io/documentation/operators/refcount.html)\
 …and then I want to ask it to start: [Connect](http://reactivex.io/documentation/operators/connect.html)

最后更新: 2020年1月14日\
