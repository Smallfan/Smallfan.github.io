---
title: 初探Flutter（一） Widget
date: 2022-09-04 18:51:55
categories: [工与匠]
tags: [Flutter]
math: true
mermaid: true
---

# 一、Widget

## 1.1 canUpdate方法

`canUpdate(...)`是一个静态方法，它主要用于在 widget 树重新`build`时复用旧的 widget。
+ 根本作用：是否用新的 widget 对象去更新旧UI树上所对应的`Element`对象的配置。
+ 判断规则：只要`newWidget`与`oldWidget`的`runtimeType`和`key`同时相等时就会用`new widget`去更新`Element`对象的配置，否则就会创建新的`Element`。

## 1.2 Flutter中的四棵树

### 1.2.1 基本联系与职责

1. 根据 Widget 树生成一个 Element 树，Element 树中的节点都继承自 `Element` 类。
2. 根据 Element 树生成 Render 树（渲染树），渲染树中的节点都继承自 `RenderObject` 类。
3. 根据渲染树生成 Layer 树，然后上屏显示，Layer 树中的节点都继承自 `Layer` 类。


+ 布局和渲染逻辑在 `Render` 树中。
+ `Element` 是 Widget 和 RenderObject 的粘合剂。

### 1.2.2 例子

```dart
Container( // 一个容器 widget
  color: Colors.blue, // 设置容器背景色
  child: Row( // 可以将子widget沿水平方向排列
    children: [
      Image.network('https://www.example.com/1.png'), // 显示图片的 widget
      const Text('A'),
    ],
  ),
);
```

<!-- more -->

注意，如果 Container 设置了背景色，Container 内部会创建一个新的 ColoredBox 来填充背景，相关逻辑如下：

```dart
if (color != null)
  current = ColoredBox(color: color!, child: current);
```

+ Image 内部会通过 RawImage 来渲染图片
+ Text 内部会通过 RichText 来渲染文本

![](https://blogpic.smallfan.top/flutter/randerTree.jpg)

### 1.2.3 四棵树的关系

1. Widget 和 Element 是一一对应的。
2. Widget 并不和 RenderObject 一一对应。比如 `StatelessWidget` 和 `StatefulWidget` 都没有对应的 RenderObject。
3. 渲染树在上屏前会生成一棵 Layer 树。

## 1.3 Widget构造函数

按照惯例，widget 的构造函数参数应使用命名参数，命名参数中的必需要传的参数要添加`required`关键字，这样有利于静态代码分析器进行检查；在继承 widget 时，第一个参数通常应该是`Key`。另外，如果 widget 需要接收子 widget ，那么`child`或`children`参数通常应被放在参数列表的最后。同样是按照惯例， widget 的属性应尽可能的被声明为`final`，防止被意外改变。

```dart
class Echo extends StatelessWidget  {
  const Echo({
    Key? key,  
    required this.text,
    this.backgroundColor = Colors.grey, //默认为灰色
  }):super(key:key);
    
  final String text;
  final Color backgroundColor;

  @override
  Widget build(BuildContext context) {
    return Center(
      child: Container(
        color: backgroundColor,
        child: Text(text),
      ),
    );
  }
}
```

# 二、StatelessWidget

## 2.1 基本概念

`StatelessWidget` 继承自`widget`类，重写了`createElement()`方法：

```dart
@override
StatelessElement createElement() => StatelessElement(this);
```

`StatelessElement` 间接继承自`Element`类。

+ 作用：`StatelessWidget`用于不需要维护状态的场景，它通常在`build`方法中通过嵌套其他 widget 来构建UI，在构建过程中会递归的构建其嵌套的 widget 。


## 2.2 Context

`build`方法有一个`context`参数，它是`BuildContext`类的一个实例，表示当前 widget 在 widget 树中的上下文，每一个 widget 都会对应一个 context 对象（因为每一个 widget 都是 widget 树上的一个节点）。

# 三、StatefulWidget

## 3.1 基本概念

和`StatelessWidget`一样，`StatefulWidget`也是继承自`widget`类，并重写了`createElement()`方法，不同的是返回的`Element` 对象并不相同；另外`StatefulWidget`类中添加了一个新的接口`createState()`。

```dart
abstract class StatefulWidget extends Widget {
  const StatefulWidget({ Key key }) : super(key: key);
    
  @override
  StatefulElement createElement() => StatefulElement(this);
    
  @protected
  State createState();
}
```

`StatefulElement`中可能会多次调用`createState()`来创建状态（State）对象。
多次调用时机：

+ 当一个 StatefulWidget 同时插入到 widget 树的多个位置时，Flutter 框架就会调用该方法为每一个位置生成一个独立的State实例，其实，本质上就是一个StatefulElement对应一个State实例。

## 3.2 三者关系

+ 当一个 StatefulWidget 同时插入到 widget 树的多个位置时，一个位置生成一个独立的State实例。
+ State 对象和`StatefulElement`具有一一对应的关系。
+ State 对象状态发生改变时，可能会重新构建新的 widget 实例。

## 3.3 State

State 中的保存的状态信息：

1. 在 widget 构建时可以被同步读取。
2. 在 widget 生命周期中可以被改变，当State被改变时，可以手动调用其setState()方法通知Flutter 框架状态发生改变，Flutter 框架在收到消息后，会重新调用其build方法重新构建 widget 树，从而达到更新UI的目的。

### 3.3.1 两个属性

1. `widget`，它表示与该 State 实例关联的 widget 实例，由Flutter 框架动态设置。注意，这种关联并非永久的，因为在应用生命周期中，UI树上的某一个节点的 widget 实例在重新构建时可能会变化，但State实例只会在第一次插入到树中时被创建，当在重新构建时，如果 widget 被修改了，Flutter 框架会动态设置State. widget 为新的 widget 实例。

2. `context`。StatefulWidget对应的 BuildContext，作用同StatelessWidget 的BuildContext。

### 3.3.2 生命周期

创建时：

```
I/flutter ( 5436): initState
I/flutter ( 5436): didChangeDependencies
I/flutter ( 5436): build
```

热重载时：

```
I/flutter ( 5436): reassemble
I/flutter ( 5436): didUpdateWidget 
I/flutter ( 5436): build
```

在 widget 树中移除，然后热重载：

```dart
 Widget build(BuildContext context) {
  //移除计数器 
  //return CounterWidget ();
  //随便返回一个Text()
  return Text("xxx");
}
```

```
I/flutter ( 5436): reassemble
I/flutter ( 5436): deactive
I/flutter ( 5436): dispose
```

### 3.3.3 initState

调用时机：当 widget 第一次插入到 widget 树时会被调用，对于每一个State对象，Flutter 框架只会调用一次该回调。
作用：通常在该回调中做一些一次性的操作，如状态初始化、订阅子树的事件通知等。

### 3.3.4 didChangeDependencies

调用时机：

+ 当State对象的依赖发生变化时会被调用。
+ 组件第一次被创建后挂载的时候（包括重创建）对应的didChangeDependencies也会被调用。

作用：当系统语言 Locale 或应用主题改变时，Flutter 框架会通知 widget 调用此回调。

### 3.3.5 build

调用时机：

1. 在调用initState()之后。
2. 在调用didUpdateWidget()之后。
3. 在调用setState()之后。
4. 在调用didChangeDependencies()之后。
5. 在State对象从树中一个位置移除后（会调用deactivate）又重新插入到树的其他位置之后。

作用：主要是用于构建 widget 子树。

### 3.3.6 didUpdateWidget

调用时机：在 widget 重新构建时，Flutter 框架会调用`widget.canUpdate`来检测 widget 树中同一位置的新旧节点，然后决定是否需要更新，如果`widget.canUpdate`返回`true`则会调用此回调。
作用：节省性能开销。

### 3.3.7 deactivate

调用时机：当 State 对象从树中被移除时。

### 3.3.8 dispose

调用时机：当 State 对象从树中被永久移除时；如果deactivate被调用后，移除的子树没有重新插入到树中则紧接着会调用dispose()方法。
作用：通常在此回调中释放资源。

## 3.4 StatefulWidget 生命周期图示

![](https://blogpic.smallfan.top/flutter/stateFlow.jpg)

## 3.5 @mustCallSuper

在继承`StatefulWidget`重写其方法时，对于包含`@mustCallSuper`标注的父类方法，都要在子类方法中调用父类方法。

# 四、在 widget 树中获取State对象

## 4.1 第一种办法：通过Context获取

```dart
class GetStateObjectRoute extends StatefulWidget {
  const GetStateObjectRoute({Key? key}) : super(key: key);

  @override
  State<GetStateObjectRoute> createState() => _GetStateObjectRouteState();
}

class _GetStateObjectRouteState extends State<GetStateObjectRoute> {
  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: Text("子树中获取State对象"),
      ),
      body: Center(
        child: Column(
          children: [
            Builder(builder: (context) {
              return ElevatedButton(
                onPressed: () {
                  // 查找父级最近的Scaffold对应的ScaffoldState对象
                  ScaffoldState _state = context.findAncestorStateOfType<ScaffoldState>()!;
                  // 打开抽屉菜单
                  _state.openDrawer();
                },
                child: Text('打开抽屉菜单1'),
              );
            }),
          ],
        ),
      ),
      drawer: Drawer(),
    );
  }
}
```

## 4.2 of静态方法的约定

一般来说，如果 StatefulWidget 的状态是私有的（不应该向外部暴露），那么我们代码中就不应该去直接获取其 State 对象；如果`StatefulWidget`的状态是希望暴露出的（通常还有一些组件的操作方法），我们则可以去直接获取其`State`对象。但是通过 context.findAncestorStateOfType 获取 StatefulWidget 的状态的方法是通用的，我们并不能在语法层面指定 StatefulWidget 的状态是否私有，所以在 Flutter 开发中便有了一个默认的约定：如果 StatefulWidget 的状态是希望暴露出的，应当在 StatefulWidget 中提供一个of 静态方法来获取其 State 对象，开发者便可直接通过该方法来获取；如果 State不希望暴露，则不提供`of`方法。这个约定在 Flutter SDK 里随处可见。所以，上面示例中的`Scaffold`也提供了一个`of`方法，我们其实是可以直接调用它的：

```dart
Builder(builder: (context) {
  return ElevatedButton(
    onPressed: () {
      // 直接通过of静态方法来获取ScaffoldState
      ScaffoldState _state = Scaffold.of(context);
      // 打开抽屉菜单
      _state.openDrawer();
    },
    child: Text('打开抽屉菜单2'),
  );
}),
```

```dart
Builder(builder: (context) {
  return ElevatedButton(
    onPressed: () {
      ScaffoldMessenger.of(context).showSnackBar(
        SnackBar(content: Text("我是SnackBar")),
      );
    },
    child: Text('显示SnackBar'),
  );
}),
```

## 4.3 第二种办法：通过GlobalKey

### 4.3.1 给目标StatefulWidget添加GlobalKey

```dart
//定义一个globalKey, 由于GlobalKey要保持全局唯一性，我们使用静态变量存储
static GlobalKey<ScaffoldState> _globalKey = GlobalKey();
...
Scaffold(
    key: _globalKey , //设置key
    ...  
)
```

### 4.3.2 通过GlobalKey来获取State对象

```dart
_globalKey.currentState.openDrawer()
```

GlobalKey 是 Flutter 提供的一种在整个 App 中引用 element 的机制。如果一个 widget 设置了`GlobalKey`，那么我们便可以通过`globalKey.currentWidget`获得该 widget 对象、`globalKey.currentElement`来获得 widget 对应的element对象，如果当前 widget 是`StatefulWidget`，则可以通过`globalKey.currentState`来获得该 widget 对应的state对象。

