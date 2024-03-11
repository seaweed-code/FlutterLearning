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
  ////注意:当Cell的高度不一致时，大幅滚动时会有性能问题 ，因为Flutter不得不计算出所有的Cell高度，然后加起来
  Widget build(BuildContext context) {
      return ListView.builder(///控件滚到屏幕外后会被重复利用，不会一次性全部创建
       // cacheExtent: 100, 超出屏幕部分缓存多少像素？上下各100像素，一般不需要设置
        itemCount: 100,
        itemBuilder: (context, i) => Text("Line----$i"),
      );
    }
  ```

- ### 使用ListView.separated构建

  ```dart
  Widget build(BuildContext context) {
      return ListView.separated(
        separatorBuilder: (context, index) => Divider(),//可以设置分割线
        cacheExtent: 100,
        itemCount: 100,
        itemBuilder: (context, i) => Text("Line----$i"),
      );
    }
  ```

- ### 使用ListView.custom构建

  

- ff