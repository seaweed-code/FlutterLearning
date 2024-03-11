## ListView使用及源码分析

如下代码，当Column内部的控件总高度超过Column的能允许的最大高度会发生溢出，这时候就需要使用ListView以允许内容滚动。

```dart
///运行会爆错：A RenderFlex overflowed by 1347 pixels on the bottom.
Widget build(BuildContext context) {
    return Column(children: [for (int i = 0; i < 100; i++) Text("Line----$i")]);
  }
```

- ### 最简单使用方法（一次性创建所有的child）

  ```dart
  ///注意：这种方法会一次性创建所有的children，一般不宜使用 
  Widget build(BuildContext context) {
      return ListView(///直接改为ListView即可
          children: [for (int i = 0; i < 100; i++) Text("Line----$i")]);
    }
  
  ///上述方法相当于如下代码：
   Widget build(BuildContext context) {
      return CustomScrollView(
        slivers: [
          ///使用SliverFixedExtentList 固定高度，可以优化滚动性能
          ///使用SliverPrototypeExtentList，设置了prototypeItem原型来动态计算固定的高度，可以优化滚动性能
          SliverList(
              delegate: SliverChildListDelegate(///SliverChildListDelegate是一次性全部创建，而不是按需加载
            [for (int i = 0; i < 100; i++) Text("Line----$i")],
          ))
        ],
      );
    }
  
  ```

  参考ListView源码：

  ```dart
  class ListView extends BoxScrollView {
    
   ///默认构造函数使用SliverChildListDelegate，一次性生成所有child，不按需加载
    ListView({
      super.key,
      super.scrollDirection,
      super.reverse,
      super.controller,
      super.primary,
      super.physics,
      super.shrinkWrap,
      super.padding,
      this.itemExtent,
      this.prototypeItem,
      bool addAutomaticKeepAlives = true,
      bool addRepaintBoundaries = true,
      bool addSemanticIndexes = true,
      super.cacheExtent,
      List<Widget> children = const <Widget>[],
      int? semanticChildCount,
      super.dragStartBehavior,
      super.keyboardDismissBehavior,
      super.restorationId,
      super.clipBehavior,
    }) : childrenDelegate = SliverChildListDelegate(///使用SliverChildListDelegate一次性创建所有child
           children,
           addAutomaticKeepAlives: addAutomaticKeepAlives,
           addRepaintBoundaries: addRepaintBoundaries,
           addSemanticIndexes: addSemanticIndexes,
         ),
         super(
           semanticChildCount: semanticChildCount ?? children.length,
         );
  
   ///Builder构造函数使用SliverChildBuilderDelegate，按需加载
    ListView.builder({
      super.key,
      super.scrollDirection,
      super.reverse,
      super.controller,
      super.primary,
      super.physics,
      super.shrinkWrap,
      super.padding,
      this.itemExtent,
      this.prototypeItem,
      required NullableIndexedWidgetBuilder itemBuilder,
      ChildIndexGetter? findChildIndexCallback,
      int? itemCount,
      bool addAutomaticKeepAlives = true,
      bool addRepaintBoundaries = true,
      bool addSemanticIndexes = true,
      super.cacheExtent,
      int? semanticChildCount,
      super.dragStartBehavior,
      super.keyboardDismissBehavior,
      super.restorationId,
      super.clipBehavior,
    }) : childrenDelegate = SliverChildBuilderDelegate(///使用SliverChildBuilderDelegate，按需创建widget
           itemBuilder,
           findChildIndexCallback: findChildIndexCallback,
           childCount: itemCount,
           addAutomaticKeepAlives: addAutomaticKeepAlives,
           addRepaintBoundaries: addRepaintBoundaries,
           addSemanticIndexes: addSemanticIndexes,
         ),
         super(
           semanticChildCount: semanticChildCount ?? itemCount,
         );
  
    ///内部自动帮我们加上分割线，如果我们有n行的话，内部加上分割线就是：2*n -1 行，其中偶数行是我们自己的，奇数行是分割线
    ListView.separated({
      super.key,
      super.scrollDirection,///水平滚动，还是垂直滚动
      super.reverse,
      super.controller,
      super.primary,
      super.physics,
      super.shrinkWrap,
      super.padding,
      required NullableIndexedWidgetBuilder itemBuilder,///每行的widget的构造方法
      ChildIndexGetter? findChildIndexCallback,
      required IndexedWidgetBuilder separatorBuilder,///分割线的构造方法
      required int itemCount,///总共多少行
      bool addAutomaticKeepAlives = true,
      bool addRepaintBoundaries = true,
      bool addSemanticIndexes = true,
      super.cacheExtent,
      super.dragStartBehavior,
      super.keyboardDismissBehavior,
      super.restorationId,
      super.clipBehavior,
    }) : itemExtent = null,
         prototypeItem = null,
         childrenDelegate = SliverChildBuilderDelegate(///跟Builder方法一样，只不过内部帮我们手动加上分割线
           (BuildContext context, int index) {
             final int itemIndex = index ~/ 2;
             if (index.isEven) {///偶数行是我们的widget
               return itemBuilder(context, itemIndex);
             }
             return separatorBuilder(context, itemIndex);//奇数行，偷偷帮我们加上分割线
           },
           findChildIndexCallback: findChildIndexCallback,
           childCount: _computeActualChildCount(itemCount),///外部行转内部实际行数：2*itemCount - 1
           addAutomaticKeepAlives: addAutomaticKeepAlives,
           addRepaintBoundaries: addRepaintBoundaries,
           addSemanticIndexes: addSemanticIndexes,
           semanticIndexCallback: (Widget widget, int index) {///偷梁换柱
             return index.isEven ? index ~/ 2 : null;///把内部的行数 转换成 外部行数
           },
         ),
         super(
           semanticChildCount: itemCount,
         );
  
   
    const ListView.custom({
      super.key,
      super.scrollDirection,
      super.reverse,
      super.controller,
      super.primary,
      super.physics,
      super.shrinkWrap,
      super.padding,
      this.itemExtent,
      this.prototypeItem,
      required this.childrenDelegate,
      super.cacheExtent,
      super.semanticChildCount,
      super.dragStartBehavior,
      super.keyboardDismissBehavior,
      super.restorationId,
      super.clipBehavior,
    }) : 
    
    ///当行高固定时，设置该字段会大大提高滚动效率
    ////设置了这个属性，内部会使用SliverFixedExtentList，固定每个Cell的高度，
    final double? itemExtent;
  
    ///当行高固定时，但需要动态计算时，设置该字段会大大提高滚动效率
    ////设置了这个属性，内部会使用SliverPrototypeExtentList，固定每个Cell的高度，
    final Widget? prototypeItem;
  
    ///如果使用默认构造函数，这个字段会被自动设置为：SliverChildListDelegate
    ///如果使用build构造函数，这个字段会自动设置为：SliverChildBuilderDelegate
    //如果使用custom构造方法，可以自定义该字段
    final SliverChildDelegate childrenDelegate;
  
    @override
    Widget buildChildLayout(BuildContext context) {
      if (itemExtent != null) {///如果ListView中设置了itemExtent固定行高，则这里就是SliverFixedExtentList
        return SliverFixedExtentList(
          delegate: childrenDelegate,
          itemExtent: itemExtent!,
        );
        ///否则如果ListView中设置了prototypeItem原型，则这里就是SliverPrototypeExtentList
      } else if (prototypeItem != null) {
        return SliverPrototypeExtentList(
          delegate: childrenDelegate,
          prototypeItem: prototypeItem!,
        );
      }
      ///如果上述两个字段都没有设置，则使用SliverList
      return SliverList(delegate: childrenDelegate);
    }
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
  ///上述方法相当于
  Widget build(BuildContext context) {
      return CustomScrollView(
        slivers: [
          SliverList(
            delegate: SliverChildBuilderDelegate(
              (context, index) {
                return Text("Line----$index");
              },
              childCount: 100,
            ),
          )
        ],
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

  1、指定itemExtent参数可以固定每行的高度，优化滚动性能。类似于iOS中UITableView设置rowHeight属性而不是用delegate返回高度。

  2、如果列表每行高度都相同但不知道具体是多少，这时候就没法使用itemExtent来优化了，可以使用prototypeItem原型返回一个widget，Flutter会动态计算其高度，然后所有行都使用此高度

  3、如果列表每行高度都不同，且需要动态计算，这时候该怎么优化呢？典型的例子就是聊天界面！

  

### CustomScrollView的使用

ListView只能做列表，GridView只能做网格类型，而CustomScrollView可以将他们混合起来一起滚动，作出非常棒的效果。