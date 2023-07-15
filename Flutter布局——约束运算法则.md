## Flutter布局——约束运算法则

##### 1、BoxConstraints:（约束）描述的是组件的width、height在一定范围内。但是flutter最终必须要能通过约束计算出每个控件的frame，我们先了解一下约束之间的简单运算

```dart
class BoxConstraints extends Constraints {
  /// Creates box constraints with the given constraints.
  const BoxConstraints({
    this.minWidth = 0.0,
    this.maxWidth = double.infinity,//∞
    this.minHeight = 0.0,
    this.maxHeight = double.infinity,//∞
  }); 
}
```

根据上述定义，我们立马能想到，此处存在3种特殊情况：

```dart
///1、宽高范围都是：0——>无穷，记为：B(0->∞),这种情况又叫做：unconstrained，无约束（默认情况，没有任何作用）
BoxConstraints({
    minWidth = 0.0,
    maxWidth = double.infinity,//∞
    minHeight = 0.0,
    maxHeight = double.infinity,//∞
  }); 

///2、宽高范围都是：0——>0，记为：B(0->0)
BoxConstraints({
    minWidth = 0.0,
    maxWidth = 0,
    minHeight = 0.0,
    maxHeight = 0,
  }); 

///3、宽高范围都是：无穷——>无穷，记为：B(∞->∞)
BoxConstraints({
    minWidth = double.infinity,
    maxWidth = double.infinity,
    minHeight =double.infinity,
    maxHeight = double.infinity,
  }); 
```

对于一个约束A，我们有如下约定：

- A的最小紧约束，记作：MinTight(A)，即宽高都取最小值

  ```dart
  MinTight(A) = BoxConstraints(
      minWidth  = A.minWidth,
      maxWidth  = A.minWidth,
      minHeight = A.minHeight,
      maxHeight = A.minHeight,
    ); 
  ```

- A的最大紧约束，记作：MaxTight(A)，即宽高都取最大值

  ```dart
  MaxTight(A) = BoxConstraints(
      minWidth  = A.maxWidth,
      maxWidth  = A.maxWidth,
      minHeight = A.maxHeight,
      maxHeight = A.maxHeight,
    ); 
  ```

约束之间的运算法则：

- enforce：【有约束A、B】把A往B里面合并，记为：A >> B，得到的约束表示为在保证满足B的情况下，尽可能满足A

  ```dart
   /// Returns new box constraints that respect the given constraints while being
    /// as close as possible to the original constraints.
  BoxConstraints enforce(BoxConstraints constraints) {
      return BoxConstraints(
        minWidth: clampDouble(minWidth, constraints.minWidth, constraints.maxWidth),
        maxWidth: clampDouble(maxWidth, constraints.minWidth, constraints.maxWidth),
        minHeight: clampDouble(minHeight, constraints.minHeight, constraints.maxHeight),
        maxHeight: clampDouble(maxHeight, constraints.minHeight, constraints.maxHeight),
      );
  }
  ///For example
  BoxConstraints A,B
  A.enforce(B) ///记为：A >> B
  ```

  由此可知最后结果一定是B的子集：

  1、当A、B无交集时，如果 A.min>B.max 则得到 MaxTight(A)，否则（A.max<B.min）得到MinTight(A)

  2、当A、B有交集时，为A和B的交集（且A>>B 和B>>A的结果相同，即为它们的交集）

  

  与上述特殊约束之间有如下规律：

  - B(0->∞)  >>  A  =  A
  - A  >>  B(0->∞)  =  A 
  - B(0->0)  >>  A   =  MinTight(A)
  - A  >>  B(0->0)    =  B(0->0) 
  - B(∞->∞)  >>  A   =  MaxTight(A)
  - A  >>  B(∞->∞)   =   B(∞->∞) 
- constrain：【有约束A，大小S】在保证不违反A约束的情况下，返回一个大小尽可能接近S的size

  ```dart
    /// Returns the size that both satisfies the constraints and is as close as
    /// possible to the given size.
    ///
    /// See also:
    ///
    ///  * [constrainDimensions], which applies the same algorithm to
    ///    separately provided widths and heights.
    Size constrain(Size size) {
      Size result = Size(constrainWidth(size.width), constrainHeight(size.height));
      assert(() {
        result = _debugPropagateDebugSize(size, result);
        return true;
      }());
      return result;
    }
  ```

- constrainDimensions：与constrain一样，推荐使用constrain

- expand：返回一个紧约束使其宽高尽可能大（或者接近指定的宽高）将此约束记为：EXPAND

  由此可知当：

  1、不传递宽高时，EXPAND = B(∞->∞)，此时：EXPAND >> A = MaxTight(A)

  2、传递宽高时，EXPAND 就是一个固定宽高的约束tight
  
  ```dart
    /// Creates box constraints that expand to fill another box constraints.
    ///
    /// If width or height is given, the constraints will require exactly the
    /// given value in the given dimension.
    const BoxConstraints.expand({
      double? width,
      double? height,
    }) : minWidth = width ?? double.infinity,
         maxWidth = width ?? double.infinity,
         minHeight = height ?? double.infinity,
         maxHeight = height ?? double.infinity;
  ```
- loosen：【有约束A】对A进行放松意为将A的minWidth、minHeight设置为0，也就是尽可能使约束“有”范围，不至于太“紧”，经常用于对紧约束（紧约束没有范围，宽高绝对固定）进行放松处理。

  由此可见：

  结果不保证一定是松约束，如果约束最大宽高本来就是0，即使缩小最小宽高到0，最后也是一个紧约束

  ```dart
    /// Returns new box constraints that remove the minimum width and height requirements.
  BoxConstraints loosen() {
      assert(debugAssertIsValid());
      return BoxConstraints(///minWidth\minHeight 不设置即为默认值0
        maxWidth: maxWidth,
        maxHeight: maxHeight,
      );
  }
  ```

  - tighten：【有约束A，大小S】在保证不违反A约束的情况下，将A变为紧约束（没有范围，width、height都绝对固定）且其size尽可能接近S

  ```dart
   /// Returns new box constraints with a tight width and/or height as close to
    /// the given width and height as possible while still respecting the original
    /// box constraints.
  BoxConstraints tighten({ double? width, double? height }) {
      return BoxConstraints(
        minWidth: width == null ? minWidth : clampDouble(width, minWidth, maxWidth),
        maxWidth: width == null ? maxWidth : clampDouble(width, minWidth, maxWidth),
        minHeight: height == null ? minHeight : clampDouble(height, minHeight, maxHeight),
        maxHeight: height == null ? maxHeight : clampDouble(height, minHeight, maxHeight),
      );
  }
  ```

- flipped：【有约束A】将约束A的宽高对调

  ```dart
   /// A box constraints with the width and height constraints flipped.
  BoxConstraints get flipped {
      return BoxConstraints(
        minWidth: minHeight,
        maxWidth: maxHeight,
        minHeight: minWidth,
        maxHeight: maxWidth,
      );
  }
  ```

- biggest：【有约束A】返回一个能满足约束A是最大size

  其本质就是返回最大宽高=Size( maxWidth, maxHeight )

  ```dart
  /// The biggest size that satisfies the constraints.
  Size get biggest => Size(constrainWidth(), constrainHeight());
  ```

- smallest：【有约束A】返回一个能满足约束A是最小size

  同上，其本质就是返回了最小宽高

  ```dart
  /// The smallest size that satisfies the constraints.
  Size get smallest => Size(constrainWidth(0.0), constrainHeight(0.0));
  ```

  

- normalize：【有约束A】对A进行标准化操作，保证没有负数，且最大值必须大于最小值

  ```dart
    /// Returns a box constraints that [isNormalized].
    ///
    /// The returned [maxWidth] is at least as large as the [minWidth]. Similarly,
    /// the returned [maxHeight] is at least as large as the [minHeight].
    BoxConstraints normalize() {
      if (isNormalized) {
        return this;
      }
      final double minWidth = this.minWidth >= 0.0 ? this.minWidth : 0.0;
      final double minHeight = this.minHeight >= 0.0 ? this.minHeight : 0.0;
      return BoxConstraints(
        minWidth: minWidth,
        maxWidth: minWidth > maxWidth ? minWidth : maxWidth,
        minHeight: minHeight,
        maxHeight: minHeight > maxHeight ? minHeight : maxHeight,
      );
    }
  ```

-  

-  

