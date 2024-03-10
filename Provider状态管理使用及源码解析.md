# Provider组件使用及源码解析

version: 6.1.2

1. ### ChangeNotifierProvider

   这是使用最多的组件，基于inheritedWidget实现。该组件携带一个数据模型，共享给下面的所有child widget 访问(read)、监听(watch)。

   由此可知：

   1、如果下面的控件监听（watch）ChangeNotifierProvider携带的数据变化，数据改变后会收到通知的

   2、如果下面的控件读取（read）ChangeNotifierProvider携带的数据，获取的是当时的数据，后续变化后是不会得到通知

2. ### Provider

   该组件携带一个数据模型，共享给下面的所有child widget 访问(read)，但不能监听(watch)。你可能要问，那有何用？为何不直接使用全局变量？好处在于当Provider在树中被移除，数据模型也会被释放。

   由此可知：

   1、如果下面的控件监听（watch）Provider携带的数据变化，是不会得到通知的

   2、如果下面的控件读取（read）Provider携带的数据，获取的是当时的数据，后续变化后是不会得到通知

3. ### ProxyProvider

   该组件功能包括 ：

   1、 Provider的功能  

   2、 可依赖上面其他模型（依赖于上面一个或多个Parrent Provider的数据。当其上面依赖的数据发送变化，自己携带的数据也会自动更新）。

   

4. ### ChangeNotifierProxyProvider

   该组件功能包括 ：

   1、 ChangeNotifierProvider的功能  

   2、 可依赖上面其他模型（依赖于上面一个或多个Parrent Provider的数据。当其上面依赖的数据发送变化，自己携带的数据也会自动更新）。

5. ### MultiProvider

6. ### Consumer

   一个Consumer就是一个监听者，可以监听上面一个或多个数据模型的变化。

7. ### Selector

   一个Selector也是一个监听者，与Consumer不同的是，可选择性监听部分变化。它本身也是全量监听了模型的变化，只不过其内部做了判断，是否需要部分刷新，底层逻辑没有用到InheritedWidget中的Aspect字段。源码如下：

   ```dart
   class Selector<A, S> extends Selector0<S> {
     Selector({
       Key? key,
       required ValueWidgetBuilder<S> builder,
       required S Function(BuildContext, A) selector,
       ShouldRebuild<S>? shouldRebuild,
       Widget? child,
     }) : super(
             key: key,
             shouldRebuild: shouldRebuild,
             builder: builder,
             selector: (context) => selector(context, Provider.of(context)),///此处最为关键，这里的context相当于中间商，它会监听模型A的任何变化，A一旦有任何改变contenx代表widget会被rebuild，但是内部会根据shouldRebuild来判断是否需要重新rebuild其child
             child: child,
           );
   }
   
   class Selector0<T> extends SingleChildStatefulWidget {
     Selector0({
       Key? key,
       required this.builder,
       required this.selector,
       ShouldRebuild<T>? shouldRebuild,
       Widget? child,
     })  : _shouldRebuild = shouldRebuild,
           super(key: key, child: child);
     
     final ValueWidgetBuilder<T> builder;///数据变化后，需要刷新则会调用重新构建子widget
     final T Function(BuildContext) selector;///对哪个字段感兴趣？就返回哪个字段
     final ShouldRebuild<T>? _shouldRebuild;///如果感兴趣的内容没有发生变化则不需要更新
   
     @override
     _Selector0State<T> createState() => _Selector0State<T>();
   }
   
   class _Selector0State<T> extends SingleChildState<Selector0<T>> {
     T? value;
     Widget? cache;
     Widget? oldWidget;
   
     @override
     Widget buildWithChild(BuildContext context, Widget? child) {
       final selected = widget.selector(context);///用户感兴趣的数据
   
       final shouldInvalidateCache = oldWidget != widget ||
           (widget._shouldRebuild != null &&
               widget._shouldRebuild!(value as T, selected)) ||
           (widget._shouldRebuild == null &&
               !const DeepCollectionEquality().equals(value, selected));///数据有没有发生变化
       if (shouldInvalidateCache) {
         value = selected;
         oldWidget = widget;
         cache = Builder(///重新构建新的Widget
           builder: (context) => widget.builder(
             context,
             selected,
             child,
           ),
         );
       }
       return cache!;
     }
     
   }
   ```

   

8. 