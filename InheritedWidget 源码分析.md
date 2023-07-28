## InheritedWidget 原理解析

#### 1、InheritedWidget虚基类

- 可以继承此类，增加一些自定义信息，以共享给所有的child。所有的InheritedWidget都会在Element Tree上被代代相传，传给他们的child。每个child widget都持有一个Map<Type,InheritedElement>保存了从自己往上的祖祖辈辈的InheritedWidget对象，所以才能快速通过ClassType找到往上最近的InheritedWidget对象，取到共享的数据
- InheritedWidget被重新创建的时候，依赖它的所有child都会收到didChangeDependencies调用

```dart
abstract class InheritedWidget extends ProxyWidget {
  const InheritedWidget({ super.key, required super.child });
  
  @override
  InheritedElement createElement() => InheritedElement(this);

///子类必须实现，当widget发生变化时，是否需要通知所有的监听者
  @protected
  bool updateShouldNotify(covariant InheritedWidget oldWidget);
}
```

#### 2、ProxyWidget

```dart
abstract class ProxyWidget extends Widget {
  const ProxyWidget({ super.key, required this.child });
  ///持有自己的child
  final Widget child;
}
```



#### 1、Element

```dart
///1、所以可以看出BuildContext本质就是该widget对应的Element对象
abstract class Element extends DiagnosticableTree implements BuildContext {

  ///祖祖辈辈传下来的所有的InheritedElement对象，Key是继承InheritedWidget的子类runtimeType，value就是该InheritedElement对象
  PersistentHashMap<Type, InheritedElement>? _inheritedElements;
  
  ///自己监听的parent InheritedElement对象（调用了dependOnInheritedWidgetOfExactType就会自动注册）
  ///没有核心用途，主要用于调试信息
  Set<InheritedElement>? _dependencies;
  
  ///从自己的祖辈widget传承来的_inheritedElements 快速找到离自己最近的InheritedWidget
   @override
  T? dependOnInheritedWidgetOfExactType<T extends InheritedWidget>({Object? aspect}) {
    final InheritedElement? ancestor = _inheritedElements == null ? null : _inheritedElements![T];
    if (ancestor != null) {///找到了，就把自己注册一下，下次就能收到parent 的通知didChangeDependencies
      return dependOnInheritedElement(ancestor, aspect: aspect) as T;
    }
    return null;
  }
  
  ///把自己监听的parent，单独保存起来，
   @override
  InheritedWidget dependOnInheritedElement(InheritedElement ancestor, { Object? aspect }) {
    _dependencies ??= HashSet<InheritedElement>();
    _dependencies!.add(ancestor);//保存起来
    ancestor.updateDependencies(this, aspect);///让parent把我也保存起来，这样它就能很方便的通知我
    return ancestor.widget as InheritedWidget;
  }
}
```

#### 2、ComponentElement

```dart
abstract class ComponentElement extends Element {
  ComponentElement(super.widget);

  Element? _child;
}
```

#### 3、ProxyElement

```dart
abstract class ProxyElement extends ComponentElement {
  /// Initializes fields for subclasses.
  ProxyElement(ProxyWidget super.widget);

  @override
  Widget build() => (widget as ProxyWidget).child;

  @override
  void update(ProxyWidget newWidget) {
    final ProxyWidget oldWidget = widget as ProxyWidget;
    assert(widget != newWidget);
    super.update(newWidget);
    assert(widget == newWidget);
    updated(oldWidget);
    rebuild(force: true);
  }

  ///当自己被重新创建时，会被调用
  ///默认情况，我们去调用所有监听自己的child的didChangeDependencies方法，
  ///子类可重写此方法，避免不必要的通知（比如新的、老的widget完全一样的情况）
  @protected
  void updated(covariant ProxyWidget oldWidget) {
    notifyClients(oldWidget);///每次更新的时候自动通知所有监听者的didChangeDependencies
  }
 
  ///需要子类InheritedElement去实现，对于InheritedElement来说调用监听者的didChangeDependencies方法
  @protected
  void notifyClients(covariant ProxyWidget oldWidget);
}
```

#### 5、InheritedElement

```dart
class InheritedElement extends ProxyElement {
  ///1、保存所有监听自己变化的child，什么时候保存的呢？什么时候去通知呢？
  ///2、通过分析知道，当需要监听自己的child调用dependOnInheritedElement时，会把它自己传递过来让我保存
  ///3、当前wiget被重新创建，调用update方法后，会遍历下面的监听者通知所有的child，我的信息已经改变
  final Map<Element, Object?> _dependents = HashMap<Element, Object?>();

  ///从上到下，将树上的所有InheritedElement，保存到字典里，一代代传递下去，如果自己也是InheritedElement，则把自己也加进去，所以每个widget都保存了自己祖祖辈辈积累的所有InheritedElement对象，好处就是用的时候可以快速查询，不用再从下往上再去找
  @override
  void _updateInheritance() {///重写了父类Element的方法，Element类默认直接保存父类的（不加自己）
    ///但因为自己是InheritedElement，所以，需要把自己加进去，以后自己的child就能看到自己了
    assert(_lifecycleState == _ElementLifecycle.active);
    final PersistentHashMap<Type, InheritedElement> incomingWidgets =
        _parent?._inheritedElements ?? const PersistentHashMap<Type, InheritedElement>.empty();
    _inheritedElements = incomingWidgets.put(widget.runtimeType, this);
  }

  ///重写父类ProxyElement的方法，此方法会被父类ProxyElement自动调用
  ///通知所有监听自己的child
  @override
  void notifyClients(InheritedWidget oldWidget) {
    for (final Element dependent in _dependents.keys) {
      ///1、确保每个dependent都是自己的child
      ///2、确保dependent的监听列表中确实有自己
      ///3、逐个调用每个监听者的didChangeDependencies 方法
      notifyDependent(oldWidget, dependent);
    }
  }
  
  ///监听我的child（也就是dependent）会通知我，让我把它保存起来，
  ///这样下次我一变化就可以快速通知它们
   @protected
  void updateDependencies(Element dependent, Object? aspect) {
    setDependencies(dependent, null);///后面的参数暂未使用，所以目前传null
  }
  
  ///把监听自己的child，通通保存到map中，value目前都是null,保留使用
   void setDependencies(Element dependent, Object? value) {
    _dependents[dependent] = value;
  }
}
```

