## Dart 语言中extends、implements、mixin的区别

三种本质上都是为了代码复用，单继承语言下如何

#### 1、extends 单继承，与所有面向对象语言一样，子类可以继承父类的所有属性、方法，且可以重写覆盖父类的方法

```dart
abstract class A {
  void test(); ///子类必须重写
  void test22() {///子类可不重写
    print("test22===");
  }
}

class B extends A {
  @override
  void test() {///父类是虚类，且此函数父类无默认实现，子类必须实现
    super.test22();///调用父类的实现
  }
}
```

#### 2、implements 接口实现

```dart
abstract class A {
  void test();
  void test22() {
    print("test22===");
  }
}

class B implements A {///子类必须实现接口的所有方法，且无法调用接口中的默认实现
  @override
  void test() {
    super.test();///Error 无法调用
  }

  @override
  void test22() {
    super.test22()///Error 无法调用
  }
}

class C extends B implements A{///OK，B已经实现了A
}
```

#### 3、mixin 混入

```dart

```

