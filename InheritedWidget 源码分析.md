## InheritedWidget 原理解析

#### 1、InheritedWidget虚基类

- 每个widget都各自持有一个PersistentHashMap<Type,InheritedElement> 对象，保存了从widget Tree中遗传下来的所有InheritedWidget对象，且代代相传。
- 可以继承此类，增加一些自定义信息，以共享给自己的child widget。由于所有的InheritedWidget都会自动传给他们的child widget。所以每个widget都能快速从Map中找到最近的InheritedWidget对象，并取到共享的数据
- 当InheritedWidget被重新创建的时候，依赖它的所有child都会收到didChangeDependencies通知，然后被rebuild

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

///祖辈传下的所有的InheritedElement对象，Key是继承InheritedWidget的子类runtimeType，value就是该InheritedElement对象
///任何context（也就是Element）都可以在Element tree从当前节点往上找InheritedElement，有了这个祖传宝贝，我们可以更快速找到
///由于key是类的类型，如果同类型的InheritedWidget类在树上被使用多次，则往下传递的过程中下面的会覆盖上面的
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
    updated(oldWidget);///参数其实并没有被使用，通知所有监听者
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
  ///1、保存所有监听自己变化的child和它关心哪些变化？，什么时候保存的呢？什么时候去通知呢？
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
  
  ///通知依赖自己的child，调用它的didChangeDependencies方法
  @protected
  void notifyDependent(covariant InheritedWidget oldWidget, Element dependent) {
    dependent.didChangeDependencies();
  }
  
  ///监听我的child（也就是dependent）会通知我，让我把它保存起来，
  ///这样下次我一变化就可以快速通知它们
   @protected
  void updateDependencies(Element dependent, Object? aspect) {
    setDependencies(dependent, null);///后面的参数暂未使用，所以目前传null,
    ///对于InheritedWidget来说，aspect都为null，表示关心所有变化，但是对于子类InheritedModel来说可不一样
  }
  
  ///把监听自己的child，通通保存到map中，value目前都是null,保留使用
   void setDependencies(Element dependent, Object? value) {
    _dependents[dependent] = value;
  }
}
```





```dart
class InheritedModelElement<T> extends InheritedElement {
  /// Creates an element that uses the given widget as its configuration.
  InheritedModelElement(InheritedModel<T> super.widget);

  ///1、某个child widget，也就是这里的dependet参数可能关心我的多个aspect，所以需要用set来保存多个aspect
  ///2、set不为nil，但是为空则表示所有aspect变化都需要通知
  ///3、set为nil表示目前还没有监听任何aspect
  @override
  void updateDependencies(Element dependent, Object? aspect) {
    ///父类里是final Map<Element, Object?> _dependents 
    //这里由于存在一对多，所以value用集合存储多个可能的aspect
    final Set<T>? dependencies = getDependencies(dependent) as Set<T>?;
    ////目前已经监听了所有的，自然就不需要再保存了，参考上面的第二点
    if (dependencies != null && dependencies.isEmpty) {
      return;
    }

    if (aspect == null) {///传过来的aspect为null，表示所有aspect都要监听，变成上面的第2点，以后所有变化都会通知
      setDependencies(dependent, HashSet<T>());
    } else {///否则，把当前感兴趣的aspect插入到集合中
      assert(aspect is T);
      setDependencies(dependent, (dependencies ?? HashSet<T>())..add(aspect as T));
    }
  }
///每个dependent监听者都可能对不同的方面感兴趣，根据保存的一一对应关系，判断是否需要通知他们
  @override
  void notifyDependent(InheritedModel<T> oldWidget, Element dependent) {
    final Set<T>? dependencies = getDependencies(dependent) as Set<T>?;///dependent所关心的哪些aspect
    ///如上面所言，set为null表示，目前没有任何监听，所以不需要处理
    if (dependencies == null) {
      return;
    }
    ///1、set不为nil，但是为空表示监听所有变化
    ///2、继承InheritedModel的子类重写了updateShouldNotifyDependent方法中，指定是否需要通知
    ///满足上面任何条件则通知监听者更新自己
    if (dependencies.isEmpty || (widget as InheritedModel<T>).updateShouldNotifyDependent(oldWidget, dependencies)) {
      dependent.didChangeDependencies();
    }
  }
}

```

```dart
abstract class InheritedModel<T> extends InheritedWidget {
  /// Creates an inherited widget that supports dependencies qualified by
  /// "aspects", i.e. a descendant widget can indicate that it should
  /// only be rebuilt if a specific aspect of the model changes.
  const InheritedModel({ super.key, required super.child });

  @override
  InheritedModelElement<T> createElement() => InheritedModelElement<T>(this);

  /// Return true if the changes between this model and [oldWidget] match any
  /// of the [dependencies].
  @protected
  bool updateShouldNotifyDependent(covariant InheritedModel<T> oldWidget, Set<T> dependencies);

  /// Returns true if this model supports the given [aspect].
  ///
  /// Returns true by default: this model supports all aspects.
  ///
  /// Subclasses may override this method to indicate that they do not support
  /// all model aspects. This is typically done when a model can be used
  /// to "shadow" some aspects of an ancestor.
  @protected
  bool isSupportedAspect(Object aspect) => true;

  ///这里考虑了一种特色情况;同一个类型的InheritedModel可能在数上会多次出现
  ///这里会从当前节点context往上找出所有的T，直到遇到第一个isSupportedAspect返回true的为止。所以数组中最后一个元素是最顶上的Widget，而且是支持该aspect的，数组前面的T，都不支持该aspect，数组第一个元素是最接近context的（也就是最底部的）
  static void _findModels<T extends InheritedModel<Object>>(BuildContext context, Object aspect, List<InheritedElement> results) {
    final InheritedElement? model = context.getElementForInheritedWidgetOfExactType<T>();
    if (model == null) {
      return;
    }

    results.add(model);

    assert(model.widget is T);
    final T modelWidget = model.widget as T;
    if (modelWidget.isSupportedAspect(aspect)) {
      return;
    }

    Element? modelParent;
    model.visitAncestorElements((Element ancestor) {
      modelParent = ancestor;
      return false;
    });
    if (modelParent == null) {
      return;
    }

    _findModels<T>(modelParent!, aspect, results);
  }

  /// Makes [context] dependent on the specified [aspect] of an [InheritedModel]
  /// of type T.
  ///
  /// When the given [aspect] of the model changes, the [context] will be
  /// rebuilt. The [updateShouldNotifyDependent] method must determine if a
  /// change in the model widget corresponds to an [aspect] value.
  ///
  /// The dependencies created by this method target all [InheritedModel] ancestors
  /// of type T up to and including the first one for which [isSupportedAspect]
  /// returns true.
  ///
  /// If [aspect] is null this method is the same as
  /// `context.dependOnInheritedWidgetOfExactType<T>()`.
  ///
  /// If no ancestor of type T exists, null is returned.
  static T? inheritFrom<T extends InheritedModel<Object>>(BuildContext context, { Object? aspect }) {
    if (aspect == null) {
      return context.dependOnInheritedWidgetOfExactType<T>();
    }

    // Create a dependency on all of the type T ancestor models up until
    // a model is found for which isSupportedAspect(aspect) is true.
    final List<InheritedElement> models = <InheritedElement>[];
    _findModels<T>(context, aspect, models);
    if (models.isEmpty) {
      return null;
    }

    final InheritedElement lastModel = models.last;
    for (final InheritedElement model in models) {
      final T value = context.dependOnInheritedElement(model, aspect: aspect) as T;
      if (model == lastModel) {
        return value;
      }
    }

    assert(false);
    return null;
  }
}
```

