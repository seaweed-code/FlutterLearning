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

