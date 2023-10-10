## Dart语言学习

本人对C、C++、Objective- C、swift语言较为熟悉，因而学习过程中，难免将Dart语言与之进行对比，总体而言，Dart语言跟它们也有一定相似性，但是更多的借鉴了它们的一些优点。下面将详细介绍Dart与其他语言不一样的地方：

- ### 变量定义

  在C、C++定义变量一般是使用类型名 + 变量名的方式，当然C++中增加auto可以使用自动类型推断，而现代编程语言例如Swift一般都可通过let、var等关键字免去手写类型名，Dart同时支持这两种方式。

  1、C、C++中定义变量

  ```c++
  int a = 9; ///C、C++、Objective-C 定义变量：类型名int + 变量名a
  bool isOk = true;
  auto b = a; ///C++方式，自动类型推断b的类型为int
  ```

  2、swift语言定义变量

  ```swift
  var a = 90;  ///变量a, 类型推断为int
  let b = 34; ///常量b, 类型推断为int 
  var ab:Int32; //变量ab，指定类型为int32
  Bool ccc; ///Error, 不接受这种方式
  ```

  3、Dart定义变量

  ```dart
  int iiii = 0; ///传统方式，OK
  var name = 'Bob';///自动推断name变量类型为String
  const cttt = 90;  ///编译时常量，必须使用编译期常量初始化
  final ff = "xxx"; ///运行时常量,只能初始化一次，如果没初始化就使用会报错
  ```

  

- ### 工厂构造方法—factory

  ```dart
  class A {
    A();
    factory A.name() {///本质就是类方法，与普通类方法不同的是，强制了返回值必须是A类型
      return A();
    }
  
    static A name1() {///普通类方法，返回值可以随意
      return A();
    }
  }
  ```

  与Objective-C中的类构造方法类似

  ```objective-c
  interface A : NSObject
  + (instancetype)name; ///类方法，必须返回A类型对象
  + (id)MethodB;///普通类方法，返回任意对象都可以
  @end
  ```

  

- ### 强类型

  - 强类型，变量被定义后，其类型完全确定，后面无法将其他类型值赋值给该变量。对C、C++而言，除非两个类型可以隐式或显示类型转换，否则无法编译。

  - 弱类型，变量被定义后，可以将其他类型赋予该变量。编译器不会报错，例如JS、Objective-C，实际运行时，如果对象真实类型没有对应方法或属性则运行时出错。

    强类型可以让编译器帮我们进行类型检查，减少出现运行时错误几率，而弱类型则相对较为灵活，但一不小心容易出现RunTime异常。Dart语言采用的是强类型，相对而言更不容易出现错误。

- ### 变量使用

  - 非空变量，必须在使用之前赋值【编译器保证—***读取之前必然已经被初始化了***，否则编译器提示错误】

    ```dart
    void fun(bool isEmpty) {
        int lineCount;//不一定在声明变量时初始化，只需在第一次用到这个变量前初始化即可
      
        ///下面的if/else两个分支，必须都对lineCount进行初始化，否则编译器会提示错误
        if (isEmpty) {
          lineCount = 0;
        } else {
          lineCount = 0;
        }
        print(lineCount);
    }
    ```

  - late 申明的变量，必须为非空变量。可延时初始化，第一次读之前还未初始化则运行奔溃

    【编译器无法保证，**需要开发人员自己保证读取前变量已初始化**，否则抛异常】

    ```dart
    void fun(bool isEmpty) {
        late int lineCount;///late类型变量，必须是非空，
      
       ///下面的if/else两个分支，有一个对lineCount进行初始化即可。都不初始化则编译器会报错
        if (isEmpty) {///如果运行时，执行到这则奔溃，因为lineCount没有初始化
          // lineCount = 0;///OK，
        } else {
          lineCount = 0;
        }
        print(lineCount);
    }
    ```

  - 可空类型变量，如果没有初始化，默认值为null

    ```dart
    int? aaa; ///没有初始化，默认值为null
    print(aaa); ///输出:null
    ```

  - final类型变量不一定要在声明时初始化，使用前必须赋值一次，而且只能赋值一次

    ```dart
    final bool isOk;
     isOk = false;
     // isOk = true;///The final variable 'isOk' can only be set once.
     print(isOk);
    ```

  - const变量，必须在声明时进行初始化，且必须用编译时常量进行初始化

    ```dart
    int a = 90;
    const xxxx = a; ///Error,不能用变量进行初始化
    const yyyy = 90;///Ok
    ```

- ### 构造函数

- ### 函数参数传递方式

  1、【值传递】对于String、Bool、int、double、Null等基础数据类型，这些类型的对象是不可修改的，一旦创建一个对象，该对象无法被修改。因而会拷贝副本，二者之间互不影响。

  2、【引用传递】其他对象（自定义的class、Map、Set、List等）拷贝的是对象地址，由于二者指向同一个对象，因而通过任一变量修改该对象的属性或状态，都会影响其他变量。

  【这里的引用传递与C++中的引用传递意义不同，更像是值传递，拷贝地址的一个副本，所以官方说法是都是值传递，理解即可】

  ```dart
  void func11(int num, List<int> array, List<int> a2) {
    num = 100;  ///修改的是副本，不影响外部变量
    array.add(34); ///通过地址修改改对象，会影响外部
  
    a2 = [9, 8];///修改地址，指向另一个数组，不影响外部
  
    print("func11");
    print(num);
    print(array);
    print(a2);
  }
  
  void main() {
    int i = 1;
    var array = [1, 6];
    var array22 = [4, 5];
  
    func11(i, array, array22);///i 拷贝一份，array、array22拷贝了地址
  
    print("-----");
    print(i); ///1
    print(array);/// [1, 6, 34]
    print(array22); //[4, 5]
  }
  
  输出结果如下；
  flutter: func11
  flutter: 100
  flutter: [1, 6, 34]
  flutter: [9, 8]
  flutter: -----
  flutter: 1
  flutter: [1, 6, 34]
  flutter: [4, 5]
  ```

  另一个例子：

  ```dart
   int old_I = 90;
    var new_I = old_I; ///把90拷贝到新的变量中
    old_I++;
    print("old---$old_I---new---$new_I");
  
    var old_Str = "uuu";
    var new_Str = old_Str;///把字符串拷贝到新的变量中
    old_Str = "iiii";
    print("old---$old_Str---new---$new_Str");
  
    {
      var old_list = [4, 5, 9];
      var new_list = old_list;///把数组对象的地址拷贝到新的变量中，二者指向的是相同的数组对象
      old_list = [78, 90];///把新的数组对象的地址赋值到该变量
      print("old---$old_list---new---$new_list");
    }
  
    var old_list = [4, 5, 9];
    var new_list = old_list;///把数组对象的地址拷贝到新的变量中，二者指向的是相同的数组对象
    old_list.clear();///把对象中元素情况，由于二者指向的是同一个数组对象，所以二个数组都为空
    print("old---$old_list---new---$new_list");
  
  输出结果如下：
  flutter: old---91---new---90
  flutter: old---iiii---new---uuu
  flutter: old---[78, 90]---new---[4, 5, 9]
  flutter: old---[]---new---[]
  ```

  

- ### 级联运算符(`..`, `?..`) 

  这个运算符是Dart语言特有的，可以让你在同一个对象上连续调用多个对象的变量或方法。比如以下代码：

  ```dart
  var paint = Paint() 
    ..color = Colors.black
    ..strokeCap = StrokeCap.round
    ..strokeWidth = 5.0;
  ```

  以上代码完全等同于如下代码：

  ```dart
  var paint = Paint();
  paint.color = Colors.black;
  paint.strokeCap = StrokeCap.round;
  paint.strokeWidth = 5.0;
  ```

  如果该对象可能为空的话，那么第一次使用级联运算符时应该使用 `?..` 以保证后续操作不会发生在null对象上。比如以下代码：

  ```dart
  querySelector('#confirm') //返回的对象可能为空
    ?..text = 'Confirm' //第一次必须使用 ?.. 如果上一句为空，后续操作不会进行
    ..classes.add('important') //后续的用不用？号无所谓，因为如果为空，执行完上一句就返回了，后续都不会执行
    ..onClick.listen((e) => window.alert('Confirmed!'))
    ..scrollIntoView();
  ```

  这段代码，等同于以下代码：

  ```dart
  var button = querySelector('#confirm');
  button?.text = 'Confirm';
  button?.classes.add('important');
  button?.onClick.listen((e) => window.alert('Confirmed!'));
  button?.scrollIntoView();
  ```

  级联运算符可以嵌套，例如：

  ```dart
  final addressBook = (AddressBookBuilder()
        ..name = 'jenny'
        ..email = 'jenny@example.com'
        ..phone = (PhoneNumberBuilder()
              ..number = '415-555-0100'
              ..label = 'home')
            .build())
      .build();
  ```

  

- ### 一切皆对象（除了Null）

  除了Null类型，其他类型都是Object的子类型。

- ### 闭包对局部变量的捕获

  1、C++的lamda表达式中，=为拷贝捕获，&为引用捕获

  2、Objective-C中普通局部变量也是拷贝捕获，__block关键字则导致代码中所有对该局部变量的访问，偷偷修改成访问堆中的变量（偷梁换柱），所以即便函数返回后再执行block也是安全的，如果是C++的引用则直接奔溃（因为局部已释放）

  ```objective-c
  void test(){
      __block int aaa = 0;
      NSLog(@"----a:%p",&aaa); ///在栈上
      dispatch_async(dispatch_get_main_queue(), ^{
          aaa = 1;
      });
      NSLog(@"+++a:%p",&aaa);///在堆上，现在的aaa已经不是原来的aaa了
  }
  
  //输入如下：
  2023-07-19 23:01:33.042980+0800 test[19863:3236537] ----a:0x16b4bd248
  2023-07-19 23:01:33.043081+0800 test[19863:3236537] +++a:0x60000237d098
  ```

  3、Dart中对局部变量没有拷贝捕获，效果看起来都是OC中的__block修饰后的效果【暂未找到理论支撑】，参考代码：

  ```dart
  void test() {
    var i = 20;///局部变量Int
    var array = [8, 9, 0];///局部变量数组
  
    //1秒后这个i行，此时函数已返回，局部变量已经不存在了按道理
    Future.delayed(Duration(milliseconds: 1000), () {//捕获了外部局部变量i和array
      print("delayed----->");
      print(i);
      print(array);
      
      i = 90; ///
      array.add(100);
    }).then((value) {//捕获了外部局部变量i和array
      print("then----->");
      print(i);
      print(array);
    });
  
    i = 12;
    array.add(888);
  
    print("test----->");
    print(i);
    print(array);
  }
  
  void main() {
    test();
    print("--end--");
  }
  ```

  上述代码的输入结果如下：【闭包没有对捕获变量copy】

  ```
  flutter: test----->
  flutter: 12
  flutter: [8, 9, 0, 888]
  flutter: --end--
  flutter: delayed----->
  flutter: 12
  flutter: [8, 9, 0, 888]
  flutter: then----->
  flutter: 90
  flutter: [8, 9, 0, 888, 100]
  ```

- ### 单继承（支持implement实现若干接口，Mixin其他class）

- ### Dart VM自动内存管理（不用程序员关心对象释放问题）

- ### 同时支持JIT（Just-In Time）、AOT（ahead of Time）

  - Just-In Time 即时编译，在程序运行时，一边解释，一边执行。执行效率低，但是移植性强。
  - Ahead of Time 提前编译，也就是在程序运行前，提前编译成当前CPU支持的机器码，执行效率高，但无法将编译好的程序移植到其他CPU体系上。

  Dart支持开发调试时使用JIT模式，而发布时使用AOT模式，鱼与熊掌兼得。

- ### 函数形参定义的多样化

- ### Future异步处理