## ListView使用及源码分析

如下代码，当Column内部的控件总高度超过Column的能允许的最大高度会发生溢出，这时候就需要使用ListView以允许内容滚动。

```dart
///运行会爆错：A RenderFlex overflowed by 1347 pixels on the bottom.
Widget build(BuildContext context) {
    return Column(children: [for (int i = 0; i < 100; i++) Text("Line----$i")]);
  }
```

- ### 最简单使用方法

  ```dart
  ///注意：这种方法会一次性创建所有的children，一般不宜使用 
  Widget build(BuildContext context) {
      return ListView(///直接改为ListView即可
          children: [for (int i = 0; i < 100; i++) Text("Line----$i")]);
    }
  ```

- ### 使用ListView.builder构建

  ```dart
    Widget build(BuildContext context) {
      return ListView.builder(///控件滚到屏幕外后会被重复利用，不会一次性全部创建
        itemCount: 100,
        itemBuilder: (context, i) => Text("Line----$i"),
      );
    }
  ```

- ### 分撒的

- ### 443