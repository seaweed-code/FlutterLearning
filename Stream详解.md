## Stream

订阅方式

1、单订阅，只允许一个订阅者，历史数据会缓存

2、广播：允许多个订阅者，数据不会缓存

#### 如何创建Stream?

- 通过async*  、yeild等关键字

  ```dart
  Stream<int> timedCounter(Duration interval, [int? maxCount]) async* {
    int i = 0;
    while (true) {
      await Future.delayed(interval);
      yield i++;
      if (i == maxCount) break;
    }
  }
  ```

- yield* 关键字

  ```dart
  void main() {
    final Stream<int> sequence = countDownByAsync(10);
    print('start');
    sequence.listen((event) => print(event)).onDone(() => print('is done'));
    print('end');
  }
  
  Stream<int> countDownByAsyncRecursive(int num) async* {
    if (num > 0) {
      yield num;///yield 生成单个数字
      yield* countDownByAsyncRecursive(num - 1);//yield* 生成一系列数字：Stream<int>
    }
  }
  ```

  特点：

  1. 异步
  2. 单订阅
  3. 被监听的时候才开始生成数字

- 通过StreamController

```dart
Stream<int> timedCounter(Duration interval, [int? maxCount]) {
  late StreamController<int> controller;
  Timer? timer;
  int counter = 0;

  void tick(_) {
    counter++;
    controller.add(counter); // Ask stream to send counter values as event.
    if (counter == maxCount) {
      timer?.cancel();
      controller.close(); // Ask stream to shut down and tell listeners.
    }
  }

  void startTimer() {
    timer = Timer.periodic(interval, tick);
  }

  void stopTimer() {
    timer?.cancel();
    timer = null;
  }

  controller = StreamController<int>(
      onListen: startTimer,///一旦有订阅时，开始
      onPause: stopTimer,///订阅中暂停时，停止
      onResume: startTimer,///订阅中恢复，重新开始
      onCancel: stopTimer);///最后一个订阅者失效时，停止

  return controller.stream;
}
```

#### 同步序列生成

- 继承**Iterable**类

  ```dart
  class StringIterable extends Iterable<String> {
    final List<String> stringList; //实际上List就是一个Iterable
  
    StringIterable(this.stringList);
  
    @override
    Iterator<String> get iterator =>
        stringList.iterator; //通过将List的iterator，赋值给iterator
  
  }
  
  //这样StringIterable就是一个特定类型String的迭代器，我们就可以使用for-in循环进行迭代
  void main() {
    var stringIterable = StringIterable([
      "Dart",
      "Java",
      "Kotlin",
      "Swift"
    ]);
    for (var value in stringIterable) {
      print('$value');
    }
  }
  
  //甚至你还可以将StringIterable结合map、where、reduce之类操作符函数之类对迭代器值进行变换
  void main() {
    var stringIterable = StringIterable([
      "Dart",
      "Java",
      "Kotlin",
      "Swift"
    ]);
    stringIterable
        .map((language) => language.length)//可以结合map一起使用，实际上本质就是Iterable对象转换，将StringIterable转换成MappedIterable
        .forEach(print);
  }
  ```

- 关键字yield 、sync*

  ```dart
  void main() {
    final numbers = getRange(1, 10);
    for (var value in numbers) {
      print('$value');
    }
  }
  
  //用于生成一个同步序列
  Iterable<int> getRange(int start, int end) sync* { //sync*告诉Dart这个函数是一个按需生产值的同步生成器函数
    for (int i = start; i <= end; i++) {
      yield i;//yield关键字有点像return，但是它是单次返回值，并不会像return直接结束整个函数
    }
  }
  ```

- yield*的使用

  ```dart
  void main() {
    final Iterable<int> sequence = countDownBySyncRecursive(10);
    print('start');
    sequence.forEach((element) => print(element));
    print('end');
  }
  
  Iterable<int> countDownBySyncRecursive(int num) sync* {
    if (num > 0) {
      yield num;///yield 单个数
      yield* countDownBySyncRecursive(num - 1);//yield* 多个数：Iterable<int>，此处是递归函数
    }
  }
  ```

  

