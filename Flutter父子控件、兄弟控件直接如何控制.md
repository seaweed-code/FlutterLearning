## Flutter父子控件、兄弟控件之间如何控制对方刷新

1. #### 通过回调函数

   优点：逻辑清晰，代码简单；缺点：一对一，需要显示参数传递

   - 子控件刷新父控件

     ```dart
     class MyHomePage extends StatefulWidget {
       const MyHomePage({super.key, required this.title});
     
       final String title;
     
       @override
       State<MyHomePage> createState() => _MyHomePageState();
     }
     
     class _MyHomePageState extends State<MyHomePage> {
       int _idx = 0;
       @override
       Widget build(BuildContext context) {
         return Scaffold(
           appBar: AppBar(
             backgroundColor: Theme.of(context).colorScheme.inversePrimary,
             title: Text(widget.title + "$_idx"),
           ),
           body: CenterWidget(
             fun: () {///传递一个函数给子控件，子控件调用该函数触发自己界面刷新
               setState(() {
                 _idx++;
               });
             },
           ),
           floatingActionButton: FloatingActionButton(
             onPressed: () {
               setState(() {
                 _idx++;
               });
             },
             tooltip: 'Increment',
             child: const Icon(Icons.add),
           ),
         );
       }
     }
     
     class CenterWidget extends StatefulWidget {
       CenterWidget({super.key, required this.fun});
     
       final VoidCallback fun;
       @override
       State<CenterWidget> createState() => _CenterWidgetState();
     }
     
     class _CenterWidgetState extends State<CenterWidget> {
       @override
       Widget build(BuildContext context) {
         return Center(
             child: Column(
           crossAxisAlignment: CrossAxisAlignment.stretch,
           mainAxisSize: MainAxisSize.min,
           children: [
             Padding(
               padding: const EdgeInsets.all(8.0),
               child: ElevatedButton(
                 onPressed: widget.fun,////子控件内部，调用外部传入的方法，触发父控件刷新
                 child: Text("Add"),
               ),
             )
           ],
         ));
       }
     }
     ```

   - 父控件刷新子控件

     ```dart
     class MyHomePage extends StatefulWidget {
       const MyHomePage({super.key, required this.title});
     
       final String title;
     
       @override
       State<MyHomePage> createState() => _MyHomePageState();
     }
     
     class _MyHomePageState extends State<MyHomePage> {
       final FunCall optChildWidget = FunCall();///由子控件去设置函数，然后自己来调用
     
       @override
       Widget build(BuildContext context) {
         return Scaffold(
           appBar: AppBar(
             backgroundColor: Theme.of(context).colorScheme.inversePrimary,
             title: Text(widget.title),
           ),
           body: CenterWidget(
             fun: optChildWidget,///传递给子控件，设置好刷新函数
           ),
           floatingActionButton: FloatingActionButton(
             onPressed: () {
               if (optChildWidget.update != null) {
                 optChildWidget.update!();///通知子控件进行刷新
               }
             },
             tooltip: 'Increment',
             child: const Icon(Icons.add),
           ),
         );
       }
     }
     
     class FunCall {
       ///child widget 回传给parrent widget
       VoidCallback? update;
     }
     
     class CenterWidget extends StatefulWidget {
       CenterWidget({super.key, required this.fun});
     
       final FunCall fun;
       @override
       State<CenterWidget> createState() => _CenterWidgetState();
     }
     
     class _CenterWidgetState extends State<CenterWidget> {
       int _counter = 0;
     
       @override
       void initState() {
         ///设置，由自己的父控件调用来刷新自己
         widget.fun.update = () {
           setState(() {
             _counter++;
           });
         };
     
         super.initState();
       }
     
       @override
       Widget build(BuildContext context) {
         return Center(
             child: Column(
           crossAxisAlignment: CrossAxisAlignment.stretch,
           mainAxisSize: MainAxisSize.min,
           children: [
             Text(
               "aaaa ${_counter}",
               textAlign: TextAlign.center,
             ),
             Padding(
               padding: const EdgeInsets.all(8.0),
               child: ElevatedButton(
                 onPressed: widget.fun.update,
                 child: Text("Add"),
               ),
             )
           ],
         ));
       }
     }
     ```

   - 兄弟节点互相刷新【原理同上】

2. #### 使用ChangeNotify发送通知配合ListenableBuilder监听通知刷新

   - 子控件刷新父控件
   - 父控件刷新子控件
   - 兄弟节点互相刷新

3. #### 使用NotificatonListener。子控件向上dispatch消息，任何父控件可以监听。

4. #### 将子控件提取成局部变量，通过变量直接操作

   - 子控件刷新父控件
   - 父控件刷新子控件
   - 兄弟节点互相刷新