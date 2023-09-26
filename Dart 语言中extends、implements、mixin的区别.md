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

1、与继承不同，非父子关系，无法调用或者重写覆盖父类方法

2、必须实现接口中的所有方法（哪怕接口中已经有默认实现）

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

在不继承该类的情况下，混入该类使得自己可以拥有该类的所有方法。其本质是：虽然代码中没有显示继承该类，但是编译器会将该类插入到当前类的继承链中，使得自己可以使用mixin类的方法。因而mixin类中的方法如果调用了super，会按照最后生成的继承链往上调用。

```dart
abstract class A {
  void test();
  void testAA() {
    print("testAA===");
  }
}

abstract class B {
  void test();
  void testBB() {
    print("testBB===");
  }
}

///M1未来可能的父类必须是A且B，由于单继承只能继承一个父类，可以是extends、implements、mixin三者任意组合
///所以在M1中可以事先调用A或者B中的方法
mixin M1 on A, B {///要想混入M1，条件是该类必须同时继承或者实现了（A、B）
  void test33() {
    test();///调用父类的test方法，可能是A中的，也有可能是B中的。看具体实现
  }

  @override
  void testBB() {///重写B中的方法
    print("M1");
    super.testBB();
  }
}

class T extends B implements A {///此处A、B的顺序调换，T的子类依然可以with M1，因为都满足了on M1的条件
  @override
  void test() {}///上面test33中调用的方法会到此处

  @override
  void testAA() {}
}

class TT extends T with M1 {}///OK，因为on M1的条件已经被父类T满足 

///允许重复with相同的类，类层次结构会多加一层
///类层次结构相当于：从上往下，父类->子类
///					T     :grandparent
///				  M1    :parent
///         M1    :son
///        TTT    :grandson
class TTT extends T with M1,M1 {}///OK
```

