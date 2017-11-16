---
title: 探寻 iPhone X 适配
date: 2017-11-16 19:08:01
tags: [iOS, iPhoneX]
---

# 前言

iPhone X 出来也有一段时间了，苹果官方和各位大神也对于 iPhone X 的刘海打理给出了很多方法。

我因个人项目在 iPhone X 的环境下运行时界面千奇百怪 (估计是代码能力太菜 - -)

所以也打算蹭一波热度，把自己对于 iPhone X 适配的一点小小的经历分享出来

# 关于iPhone X

## 一个有刘海的屏幕

iPhone X 的机型的屏幕尺寸达到了 5.8英寸，而机子的宽度和 iPhone 8 一致，在垂直方向上却多出了 145pt，这意味着可以显示的界面又一次变“长”了。

值得注意的是 iPhone X 的顶部是有圆角和刘海的，所以我们在设计页面的时候还需要避免内容被顶部的圆角还有刘海挡住。

![CGRect(0,0,200,200)](/img/圆角遮挡.png)

## 全新的界面理念

### Status Bar

iPhone X 的 Status Bar 高度比之前的 20pt 高了 22pt。也就是说现在的高度为 44pt。(还好我没把之前的项目写死在 20pt，不然估计现在心态崩了)

### 屏幕底部

自从 iPhone X 没有了 Home 键，苹果在底部划了一个 34pt 高的区域( Home Indicator )来预留给底部手势操作

如果我们的底部是 TabBar，那 Home Indicator 会延展你的 barTintColor，如果没有的话就会显示为透明的。

### SafeArea

iOS 11 废弃了 iOS 7 之后出现的 topLayoutGuide/bottomLayoutGuide，而用 SafeLayoutGuide 取而代之。

我们的 UI 都应该在 SafeArea 之内，以便不被各种 bar(NavigationBar、ToolBar、Tabbar、StatusBar)遮挡

![SafeArea](/img/SafeArea.png)

若我们使用 AutoLayout，并开启了 safeAreaLayoutGuide，布局会自动加上 safeLayoutGuide，也就是自动将我们的布局限制在 SafeArea 之中。当然，我们也可以使用 self.additionalSafeAreaInsets 来增加 Guide 的区域。

在我们用 Masonry 进行布局时，亦需要进行适配，我们可以把

`make.bottom.equalTo(self.view.mas_bottom;)`

和

`make.top.equalTo(self.view.mas_top);`

改为

```objective-c
if (@available (iOS 11.0, *)){
  make.bottom.equalTo(self.view.mas_safeAreaLayoutGuideBottom);
  make.bottom.equalTo(self.view.mas_safeAreaLayoutGuideTop);
}else {
  make.bottom.equalTo(self.view.mas_bottom);
  make.bottom.equalTo(self.view.mas_top);
}
```



### 其他

图片的 Aspect Ratio 的表现也不同了

因为刘海两侧的区域都能响应手势，所以最好不要和自己界面的手势发生冲突。

如果我们使用 Images.xcassets 文件夹下的 LaunchImage 作为启动图，就必须单独制作一张 1125px × 2436px (375pt × 812pt @3x) 图片，不然沿用 iPhone 7 启动图尺寸会出现上下有黑色区域的情况。

如果我们以一个界面启动，那我们也需要重新做一张图来避免图片因为屏幕大小改变而拉伸过度。



# 参考

[美团点评技术团队 - iPhone X 刘海打理指北](https://tech.meituan.com/iPhoneX%E5%88%98%E6%B5%B7%E6%89%93%E7%90%86%E6%8C%87%E5%8C%97.html)