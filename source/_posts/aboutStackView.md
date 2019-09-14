---
title: 拥抱 UIStackView
date: 2019-09-01 14:31:36
tags: [iOS]
---

# 前言

　　作为 iOS 开发者，难免会遇到一些 “一个 View 需要基于另一个 View 的消失或出现变更约束”的需求。我之前的做法是选择用 `Masnory` 将两个约束手动设置成 `deactivate` 和 `activate`，在需要的时候变更属性就可以达到。或者使用约束优先级来做，不过约束优先级就不能只依靠 `hidden` 属性了，需要的可能是 `removefromsuperView`。

　　苹果在 iOS 9 之后推出了 `UIStackView`，它可以实现上述需求的自动化处理，让我们的代码更加简洁易读。之前因为开发者们向下兼容版本的原则导致运用的并不太多，但是随着现在 iOS 13 的发布，开发者们也都开始逐步放弃兼容 iOS 8了。这就意味着我们可以随心所欲的在我们的项目中使用 `UIStackView` 了 。

<!-- more -->

# 关于 UIStackView

　　`UIStackView`是`UIView`的子类，只有它并不能达到呈现视图的效果，我们要做的是把控件放入`stackView`里，让它来帮助我们管理这些控件的约束。

　　`stackView`有两种方式可以使用，一就是使用`StoryBoard`拖入一个`stackView`，二是使用代码布局加入`UIStackView`，并`addArrangeView`控件来达到约束效果。

　　因为平时使用的代码进行布局而非`StoryBoard`，所以本文会以代码的角度来走进`UIStackView`



## UIStackView 的属性


* Axis: 将`stackView`里的子视图设置成垂直布局或水平布局
* Alignment: 子视图对齐方式
* Distribution: 子视图的分布比例
* Spacing: 子视图间距

![stackView属性](/img/stackView/stackViewAttribute.png)

### Axis

　　`Axis` 属性从他的名字和他的值`vertical`和`horizontal`就可以看出他是决定`stackView`轴线的属性。

* vertical: 使 **stackView** 里的对象垂直排列

* horizontal: 使 **stackView** 里的对象水平排列



### Alignment

　　`Alignment`属性的值就有些多了，根据设置它的值，来决定子视图的摆放样式。



**Fill**

　　`Fill`属性使子视图填满`stackView`，`Vertical`模式时则是子视图宽填满，`Horizontal`模式的时候是子视图高填满。

![Fill](/img/stackView/Alignment_Fill.png)



**Leading**

　　使`Vertical`模式下的子视图向左对齐，相当于`Horizontal`模式下的`top`对齐模式旋转90°。

![Leading](/img/stackView/Alignment_Leading.png)



**Top**

　　使`Horizontal`模式下的子视图顶部对齐。

![Top](/img/stackView/Alignment_Top.png)



**FirstBaseline**

　　使子视图根据第一个子视图文字的第一行对齐（只在`Horizontal`模式下生效）

![FirstBaseline](/img/stackView/Alignment_FirstBaseLine.png)



**LastBaseline**

　　使子视图根据首个子视图文字的最后一行对齐（只在`Horizontal`模式下生效）

![LastBaseLine](/img/stackView/Alignment_LastBaseLine.png)



**Center**

　　使子视图中央对齐

![Center](/img/stackView/Alignment_Center.png)



**Trailing**

　　`Vertical`模式下的子视图尾部（右侧）对齐

![Trailing](/img/stackView/Alignment_Trailing.png)



**Bottom**

　　`Horizontal`模式下子视图尾部（下方）对齐

![Alignment_Bottom](/img/stackView/Alignment_Bottom.png)



### Distribution

　　属性作用于子视图上，用于子视图的布局



**Fill**

　　使子视图的宽/高（根据`Axis`决定）充满`stackView`，它会优先压缩`hugging`优先级低的子视图。

![DistributionFill](/img/stackView/DistributionFill.png)



**FillEqually**

　　使子视图的宽/高（根据`Axis`决定）相等排列，所有子视图都会被拉伸或压缩。

![DistributionFillEqually](/img/stackView/DistributionFillEqually.png)



**FillPropoetionally**

　　根据子视图的比例来拉伸或压缩宽/高。如`Horizontal`下的三个子视图的宽度比是 1:2:3 ，那根据`stackView`的大小，子视图会被拉伸或者压缩，但是他们的宽度比始终是 1:2:3

![DistributionFillProportionally](/img/stackView/DistributionFillProportionally.png)



**EqualSpacing**

　　使子视图宽高不变，中间等距排列（自动计算间距使得子控件充满`stackView`）

![DistributionEqualSpacing](/img/stackView/DistributionEqualSpacing.png)



**EqualCentring**

　　使子视图的中心点之间的距离保持一致，有点类似于`EqualSpacing`

![DistributionEqualCentering](/img/stackView/DistributionEqualCentering.png)



### Spacing

　　控制子视图之间的间隔，会优先生效。

　　如在`FillEqually`情况下，子视图设置了`Spacing`，则子视图之间会有固定间隔，且子视图的宽/高被压缩。



## UIStackView的使用

　　`stackView`的使用非常方便，只需要在初始化的时候设置上文的属性，并且将子视图`addArrangedSubview`进`stackView`就可以实现布局了。当一个子视图被`remove`或`hidden`时，在`stackView`里的其他子视图就会自动调整。

```objc
// 示例
UIStackView *stackView = [[UIStackView alloc] init];
stackView.alignment = UIStackViewAlignmentLeading;
stackView.axis = UILayoutConstraintAxisVertical;
stackView.spacing = 2;
[stackView addArrangedSubView:View1];
[stackView addArrangedSubView:View2]; // View2会显示在View1的下方且间隔为2
```

　　熟悉了`stackView`的使用之后，我们会发现我们大多数用`stackView`都是用于解决等距排列的需求。

　　但我们偶尔还会遇到一些子视图之间间隔不等但还需要自动排列控件的情况，这种需求我们第一选择可能不会使用`stackView`。

　　事实上，`stackView`里还有一个`setCustomSpacing:afterView:`方法，用来设置某个特定 View 以后的间隔，来达到控件不同间隔的效果。但是这个方法只在`iOS 11`以上的系统才能生效，所以现有情况下，我们可以使用空白的 View 去填充一些空白，来达到子视图之间间隔不相同的效果。



# 后记

　　`UIStackView`是苹果公司很早就退出的一套约束管理控件，但是因为兼容系统版本的问题，我们迟迟没有全面用上。但是现在已经是`iOS 13`的时代了，我们完全可以张开双手，去拥抱`stackView`，让我们的代码更加简洁有效。

　　同时希望`Swift UI`普及的那一天快一点到来。

