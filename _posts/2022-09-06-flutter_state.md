---
title: 初探Flutter（二） 状态管理
date: 2022-09-06 22:52:55
categories: [工与匠, Flutter]
tags: [Flutter基础]
math: true
mermaid: true
---

# 一、三种常见的状态管理

+ Widget 管理自己的状态。
+ Widget 管理子 Widget 状态。
+ 混合管理（父 Widget 和子 Widget 都管理状态）。

## 1.1 Widget 管理自己的状态

### 1.1.1 适用场景

如果状态是有关界面外观效果的，例如颜色、动画，那么状态最好由 Widget 本身来管理。

### 1.1.2 例子

```dart
class TapboxA extends StatefulWidget {
  TapboxA({Key? key}) : super(key: key);

  @override
  _TapboxAState createState() => _TapboxAState();
}

class _TapboxAState extends State<TapboxA> {
  bool _active = false;

  void _handleTap() {
    setState(() {
      _active = !_active;
    });
  }

  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: _handleTap,
      child: Container(
        child: Center(
          child: Text(
            _active ? 'Active' : 'Inactive',
            style: TextStyle(fontSize: 32.0, color: Colors.white),
          ),
        ),
        width: 200.0,
        height: 200.0,
        decoration: BoxDecoration(
          color: _active ? Colors.lightGreen[700] : Colors.grey[600],
        ),
      ),
    );
  }
}
```

<!-- more -->

## 1.2 Widget 管理子 Widget 状态

### 1.2.1 适用场景

如果状态是用户数据，如复选框的选中状态、滑块的位置，则该状态最好由父 Widget 管理。

### 1.2.2 例子

```dart
//------------------------ ParentWidget --------------------------------

class ParentWidget extends StatefulWidget {
  @override
  _ParentWidgetState createState() => _ParentWidgetState();
}

class _ParentWidgetState extends State<ParentWidget> {
  bool _active = false;

  void _handleTapboxChanged(bool newValue) {
    setState(() {
      _active = newValue;
    });
  }

  @override
  Widget build(BuildContext context) {
    return Container(
      child: TapboxB(
        active: _active,
        onChanged: _handleTapboxChanged,
      ),
    );
  }
}

//------------------------- TapboxB ----------------------------------

class TapboxB extends StatelessWidget {
  TapboxB({Key? key, this.active: false, required this.onChanged})
      : super(key: key);

  final bool active;
  final ValueChanged<bool> onChanged;

  void _handleTap() {
    onChanged(!active);
  }

  Widget build(BuildContext context) {
    return GestureDetector(
      onTap: _handleTap,
      child: Container(
        child: Center(
          child: Text(
            active ? 'Active' : 'Inactive',
            style: TextStyle(fontSize: 32.0, color: Colors.white),
          ),
        ),
        width: 200.0,
        height: 200.0,
        decoration: BoxDecoration(
          color: active ? Colors.lightGreen[700] : Colors.grey[600],
        ),
      ),
    );
  }
}
```

## 1.3 混合管理（父 Widget 和子 Widget 都管理状态）

### 1.3.1 适用场景

如果某一个状态是不同 Widget 共享的则最好由它们共同的父 Widget 管理。

### 1.3.2 例子

`_ParentWidgetStateC`类:

+ 管理`_active`状态。
+ 实现 `_handleTapboxChanged()` ，当盒子被点击时调用。
+ 当点击盒子并且`_active`状态改变时调用`setState()`更新UI。

`_TapboxCState`对象:

+ 管理`_highlight`状态。
+ `GestureDetector`监听所有tap事件。当用户点下时，它添加高亮（深绿色边框）；当用户释放时，会移除高亮。
+ 当按下、抬起、或者取消点击时更新`_highlight`状态，调用`setState()`更新UI。
+ 当点击时，将状态的改变传递给父组件。

```dart
//---------------------------- ParentWidget ----------------------------

class ParentWidgetC extends StatefulWidget {
  @override
  _ParentWidgetCState createState() => _ParentWidgetCState();
}

class _ParentWidgetCState extends State<ParentWidgetC> {
  bool _active = false;

  void _handleTapboxChanged(bool newValue) {
    setState(() {
      _active = newValue;
    });
  }

  @override
  Widget build(BuildContext context) {
    return Container(
      child: TapboxC(
        active: _active,
        onChanged: _handleTapboxChanged,
      ),
    );
  }
}

//----------------------------- TapboxC ------------------------------

class TapboxC extends StatefulWidget {
  TapboxC({Key? key, this.active: false, required this.onChanged})
      : super(key: key);

  final bool active;
  final ValueChanged<bool> onChanged;
  
  @override
  _TapboxCState createState() => _TapboxCState();
}

class _TapboxCState extends State<TapboxC> {
  bool _highlight = false;

  void _handleTapDown(TapDownDetails details) {
    setState(() {
      _highlight = true;
    });
  }

  void _handleTapUp(TapUpDetails details) {
    setState(() {
      _highlight = false;
    });
  }

  void _handleTapCancel() {
    setState(() {
      _highlight = false;
    });
  }

  void _handleTap() {
    widget.onChanged(!widget.active);
  }

  @override
  Widget build(BuildContext context) {
    // 在按下时添加绿色边框，当抬起时，取消高亮  
    return GestureDetector(
      onTapDown: _handleTapDown, // 处理按下事件
      onTapUp: _handleTapUp, // 处理抬起事件
      onTap: _handleTap,
      onTapCancel: _handleTapCancel,
      child: Container(
        child: Center(
          child: Text(
            widget.active ? 'Active' : 'Inactive',
            style: TextStyle(fontSize: 32.0, color: Colors.white),
          ),
        ),
        width: 200.0,
        height: 200.0,
        decoration: BoxDecoration(
          color: widget.active ? Colors.lightGreen[700] : Colors.grey[600],
          border: _highlight
              ? Border.all(
                  color: Colors.teal[700],
                  width: 10.0,
                )
              : null,
        ),
      ),
    );
  }
}
```

## 1.4 通用选型法则

在 Widget 内部管理状态封装性会好一些，而在父 Widget 中管理会比较灵活。
如果不确定到底该怎么管理状态，那么推荐的首选是在父 Widget 中管理（灵活会显得更重要一些）。


# 二、全局状态管理

实际例子：我们有一个设置页，里面可以设置应用的语言，我们为了让设置实时生效，我们期望在语言状态发生改变时，App中依赖应用语言的组件能够重新 build 一下，但这些依赖应用语言的组件和设置页并不在一起，所以这种情况用上面的方法很难管理。这时，正确的做法是通过一个全局状态管理器来处理这种相距较远的组件之间的通信。

1. 实现一个全局的事件总线，将语言状态改变对应为一个事件，然后在APP中依赖应用语言的组件的`initState` 方法中订阅语言改变的事件。当用户在设置页切换语言后，我们发布语言改变事件，而订阅了此事件的组件就会收到通知，收到通知后调用`setState(...)`方法重新`build`一下自身即可。
2. 使用一些专门用于状态管理的包，如 Provider、Redux，读者可以在 pub 上查看其详细信息。