---
title: 适配iPhoneX & iOS11
date: 2018-01-02 18:33:55
tags:
---

## 一、Screen Size

iPhoneX 的屏幕尺寸为 375pt × 812pt @3x，像素为 1125px × 2436px。可以通过判断屏幕的高度来判断设备是否是 iPhoneX，可以在全局宏定义中添加判断设备的宏定义（横竖屏通用）：

```objc
#define IS_IPHONE_X     (( fabs((double)[[UIScreen mainScreen] bounds].size.height - (double)812) < DBL_EPSILON ) || (fabs((double)[[UIScreen mainScreen] bounds].size.width - (double)812) < DBL_EPSILON ))
```

如果在 iPhoneX模拟器运行现有 app，出现上下屏幕没填充满的情况时，说明 app 没有适合 iPhoneX 尺寸的启动图，因此，需要添加一张 1125px × 2436px（@3x）的启动图，或者在项目中添加 LaunchScreen.xib，然后在项目的 target 中，设置启动 Launch Screen File 为 LaunchScreen.xib。

## 二、safe Area

官方指出：

> When designing for iPhone X, you must ensure that layouts fill the screen and aren't obscured by the device's rounded corners, sensor housing, or the indicator for accessing the Home screen.

当我们在设计 iPhoneX app 的时候，必须确保布局充满屏幕，并且布局不会被设备的圆角、传感器外壳或者用于访问主屏幕的指示灯遮挡住。因此，苹果提出了`safe area（安全区）`的概念，就是上述可能遮挡界面的区域以外的区域被定义为安全区。

![竖屏](https://user-gold-cdn.xitu.io/2018/1/2/160b5085834364c7?w=120&h=236&f=png&s=4062)
![横屏](https://user-gold-cdn.xitu.io/2018/1/2/160b508586c91670?w=252&h=120&f=png&s=3911)

为了尽可能的使布局和手势等不被圆角和传感器遮挡，竖屏情况下，苹果官方建议的安全区大小为上图（竖屏），指定的状态栏高度为 44pt，下方指示灯处的高度为 34pt；横屏情况下为上图（横屏）所示，上下安全边距分别为 0pt/21pt，左右安全边距为 44pt/34pt，如果使用了`UINavigationBar`和`UITabBar`，安全区的上边缘会变成导航栏下边缘的位置，如果是自定义的`navigationBar`，并且还继承于`UIView`，就需要手动修改状态栏的高度，在我们的项目中，状态栏的高度是用的全局宏定义，因此，修改状态栏高度的宏定义为：

```objc
#define STATUS_HEIGHT   (IS_IPHONE_X?44:20)
```

增加安全区域下面的区域的高度宏定义为：

```objc
#define BOTTOM_SAFEAREA_HEIGHT (IS_IPHONE_X? 34 : 0)
```

如果你同时也用了自定义的`UITabBar`那么就需要修改`TABBAR_HEIGHT `的宏定义为：

```objc
#define TABBAR_HEIGHT   (IS_IPHONE_X? (49 + 34) : 49)
```

当需要将整个界面最下方的控件上移或者改变中间滚动视图的高度的时候，使用`BOTTOM_SAFEAREA_HEIGHT `这个宏，方便后期的统一维护。因为基本上每一个界面都需要下方留白，因此在`BaseViewController`添加属性：

```objc
@property (nonatomic, strong) UIView *areaBelowSafeArea;
```

并且统一添加到`view`上：

```objc
#warning 背景色待定
if (IS_IPHONE_X) {
   self.areaBelowSafeArea = [[UIView alloc] initWithFrame:CGRectMake(0, SCREEN_HEIGHT - BOTTOM_SAFEAREA_HEIGHT, SCREEN_WIDTH, BOTTOM_SAFEAREA_HEIGHT)];
   // 尽量使用约束布局
   self.areaBelowSafeArea.backgroundColor = DefaultTabBarBackgroundColor;
   [self.view addSubview:self.areaBelowSafeArea];
 }
```

也可以在特定的`viewController`中，自定义它的样式。

```objc
self.areaBelowSafeArea.backgroundColor = XXX;
```

更详细：[Designing for iPhone X](https://developer.apple.com/videos/play/fall2017/801/)

## 三、UIScrollView及其子类

在 iOS11 中，决定滚动视图的内容和边缘距离的属性改为`adjustedContentInset `，而不是原来的`contentInsets`，在 iOS11 之前，`UIViewController`有一个`automaticallyAdjustsScrollViewInsets`属性，并且默认值为`YES`，这个属性的作用为，当`scrollView`为控制器根视图的最上层子视图时，如果这个控制器被嵌入到`UINavigationController`和`UITabBarController`中，那么，它的`contentInsets`会自动设置为`(64,0,49,0)`；这个属性会使滚动视图中的内容不被导航栏和`tabBar`遮挡。

iOS11 使用了`safeAreaInsets`的新属性，这个属性的作用就是规定了视图的安全区的四个边到屏幕的四个边的距离，例如在 iPhoneX 上，如果没使用或者隐藏了`UINavigationBar`，则`safeAreaInsets = (44,0,0,34)`，如果既使用了`UINavigationBar`，又使用了`UITabBar`，则`safeAreaInset = (88,0,0,34+49)`。也可以使用`additionalSafeAreaInsets `属性来为系统默认的`safeAreaInsets`添加 insets，比如，`safeAreaInsets = (44,0,0,34)`,设置`additionalSafeAreaInsets = UIEdgeInsetsMake(-44, 0, 0, 0);`，那么实际上的安全区域到屏幕边缘的insets为`(0,0,0,34)`。

`adjustedContentInset`属性的值的确定由 iOS11 API 提供的新的枚举变量`contentInsetAdjustmentBehavior`决定。这个属性的类型定义为：

```objc
typedef NS_ENUM(NSInteger, UIScrollViewContentInsetAdjustmentBehavior) {
    UIScrollViewContentInsetAdjustmentAutomatic, // Similar to .scrollableAxes, but for backward compatibility will also adjust the top & bottom contentInset when the scroll view is owned by a view controller with automaticallyAdjustsScrollViewInsets = YES inside a navigation controller, regardless of whether the scroll view is scrollable
    UIScrollViewContentInsetAdjustmentScrollableAxes, // Edges for scrollable axes are adjusted (i.e., contentSize.width/height > frame.size.width/height or alwaysBounceHorizontal/Vertical = YES)
    UIScrollViewContentInsetAdjustmentNever, // contentInset is not adjusted
    UIScrollViewContentInsetAdjustmentAlways, // contentInset is always adjusted by the scroll view's safeAreaInsets
} API_AVAILABLE(ios(11.0),tvos(11.0));
```

四个枚举值的意义分别为：

`UIScrollViewContentInsetAdjustmentAutomatic`：

> Content is always adjusted vertically when the scroll view is the content view of a view controller that is currently displayed by a navigation or tab bar controller. If the scroll view is horizontally scrollable, the horizontal content offset is also adjusted when there are nonzero safe area insets.

当滚动视图的父视图所在的控制器嵌入导航控制器和标签控制器的时候，滚动视图的内容总会调整垂直方向上的 insets，如果滚动视图允许水平方向上可滚动，则当水平方向上的安全区 insets 不为 0 的时候，也会调整水平方向上的 insets。即：`adjustedContentInset = safeAreaInsets + contentInsets`，其中`contentInsets`为我们设置的滚动视图的`contentInsets`，下同。如下代码，

```objc
self.scroll.contentInset = UIEdgeInsetsMake(100, 0, -34, 10);
if (@available(iOS 11.0, *)) {
    self.scroll.contentInsetAdjustmentBehavior = UIScrollViewContentInsetAdjustmentAutomatic;
}else {
    self.automaticallyAdjustsScrollViewInsets = YES;
}
```

添加导航栏的情况下，在 iPhoneX 设备上运行打印的 log 为：

```objc
2017-10-20 11:48:56.664048+0800 TestIphoneX[81880:2541000] contentInset:{100, 0, -34, 10}
2017-10-20 11:48:56.664916+0800 TestIphoneX[81880:2541000] adjustedContentInset:{188, 0, 0, 10}
2017-10-20 11:48:56.665220+0800 TestIphoneX[81880:2541000] safeAreaInset:{88, 0, 34, 0}
```

`UIScrollViewContentInsetAdjustmentScrollableAxes`：

> The top and bottom insets include the safe area inset values when the vertical content size is greater than the height of the scroll view itself. The top and bottom insets are also adjusted when the alwaysBounceVertical property is YES. Similarly, the left and right insets include the safe area insets when the horizontal content size is greater than the width of the scroll view.

`adjustedContentInset = safeAreaInsets + contentInsets`，它的成立依赖于滚动轴，当垂直方向上的`contentSize`大于滚动视图的高度时，那么垂直方向上的 insets 就由`safeAreaInsets + contentInsets`决定，水平方向上同理。

`UIScrollViewContentInsetAdjustmentNever`：

> Do not adjust the scroll view insets.

顾名思义，`adjustedContentInset = contentInsets`

`UIScrollViewContentInsetAdjustmentAlways`：

> Always include the safe area insets in the content adjustment.

顾名思义，`adjustedContentInset = safeAreaInsets + contentInsets`

由于我们的 APP 没用系统的导航控制器，但是我们用了系统的标签控制器，所以在项目中会存在 iOS11 下滚动视图的位置不对的情况，那么就可能是因为它的`adjustedContentInset = safeAreaInsets + contentInsets`造成的，可以这样解决：

```objc
if (@available(iOS 11.0, *)) {
    self.scroll.contentInsetAdjustmentBehavior = UIScrollViewContentInsetAdjustmentNever;
}else {
    self.automaticallyAdjustsScrollViewInsets = NO;
}
```

## 四、UITableView sectionHeader 和 sectionFooter高度问题

iOS11 中 `UITableView`的 sectionHeader 和 sectionFooter 也启用了 `self-sizing`，即通过估算的高度乘以个数来确定`tableView`的`contenSize`的估算值，然后随着滚动展示 section 和 cell 的过程中更新它的`contenSize`，iOS11 之前只有 cell 是采用的这个机制，iOS11中 sectionHeader 和 sectionFooter 也采用了这个机制，并且，如果只实现了

```objc
- (CGFloat)tableView:(UITableView *)tableView heightForHeaderInSection:(NSInteger)section;
```

没实现

```objc
- (nullable UIView *)tableView:(UITableView *)tableView viewForHeaderInSection:(NSInteger)section;
```

那么系统就会直接采用估算的高度，而不是`heightForHeaderInSection`方法中设置的高度，也就是此时的`sectionHeight`为

```objc
- (CGFloat)tableView:(UITableView *)tableView estimatedHeightForHeaderInSection:(NSInteger)section
```

方法，或者`estimatedSectionHeaderHeight`设置的估算高度，如果没有设置估算高度，则系统默认为`UITableViewAutomaticDimension`。

所以必须同时实现`heightForHeaderInSection`和`viewForHeaderInSection `方法，可以返回`[UIView new]`，但是不能不实现。或者只实现`heightForHeaderInSection`方法，并且设置`estimatedSectionHeaderHeight`为 0 来关闭估算机制。

注：如果在 iOS11 中，使用了 self-sizing cell，并且使用了上拉加载更多，并且使用了高度自适应的方式计算 cell 的高度，那么上拉加载更多的时候会发现 tableView 会跳动一下或者滚动一段距离，什么原因呢，这里解释一下可能是由于：假如一个列表有10个 cell，你设置的估算高度是 80，那么整个列表的估算高度为`10 * 80 = 800`，但是实际高度不是 800，假如是 1000，那么当滚动到最下方的时候，此时的`contentOffSet = 1000`，然后上拉再加载 10条数据，此时会调用`- (void)reloadData;`方法，此时，列表的高度仍然会重新使用估算高度计算，`80 * 20 = 1600`，而`contentOffSet = 1000`，这个位置已经不是刚才的第 10条数据了，而是第`1000 / 80= 12.5`条数据了，因此会造成加载更多的时候数据衔接不上的问题。你可能需要设置`estimatedRowHeight = 0 `来关闭它的估算高度解决这个问题，但是如果你非要开启它的估算高度来使 cell 中的约束自适应高度的话，可以通过这种方式计算高度：

```objc
- (CGFloat)tableView:(UITableView *)tableView heightForRowAtIndexPath:(NSIndexPath *)indexPath
{
    static DisplayCell *cell;//‘static’将cell存储在静态存储区，这里创建的cell仅用来计算高度，因此，内存中只有一份就可以了，因为此方法会调用多次，每次都创建的话即会耗费时间也会耗费空间。
    if (!cell) {
        cell = [[[NSBundle mainBundle] loadNibNamed:kCellNibName owner:self options:nil] lastObject];
    }
    cell.displayLab.text = self.data[indexPath.row];//给cell赋值，赋值是为了通过内容计算高度
    CGFloat height = [cell systemLayoutSizeFittingSize:UILayoutFittingCompressedSize].height;
    return height;
}
```

另外，如果 cell 中使用了多行 label 的话，注意设置它的换行宽度：
`self.displayLab.preferredMaxLayoutWidth = XXX;//XXX应该和你约束的label宽度相同`

## 五、Tips

### 1、每个界面中的控件的位置

如果项目中用 frame 布局的控件较多，很多控件的位置依赖于`self.view`的顶部和底部，由于状态栏和底部空间的调整就会造成一部分控件的位置发生变化，修改过程中应该注意和线上 APP 比对。建议能用约束的就别用 frame，依赖上下控件的位置比依赖屏幕的边缘和宽高更好维护一些。autolayout 并不影响写动画！

### 2、重新布局

项目中有些控件的位置会因为响应事件、动画和数据请求等重新布局，因此应该特别注意的地方就是重新布局后控件的位置是否和线上项目一致。另外，还有一些初始化时隐藏的控件，由于某些条件发生后才展示，也要注意其布局。

### 3、全屏显示

全屏显示和横屏模式下的界面，注意横屏之后下方的感应器在安全区之外。

### 4、LaunchuImage

像素为：1125 * 2436
并在LaunchImage中的Contents.json文件中增加 JSON：

```json
{
    "extent" : "full-screen",
    "idiom" : "iphone",
    "subtype" : "2436h",
    "filename" : "图片名字.png",
    "minimum-system-version" : "11.0",
    "orientation" : "portrait",
    "scale" : "3x"
}
```

### 5、定位

在 iOS 11 中必须支持 When In Use 授权模式（NSLocationWhenInUseUsageDescription），在 iOS 11 中，为了避免开发者只提供请求 Always 授权模式这种情况，加入此限制，如果不提供When In Use 授权模式，那么 Always 相关授权模式也无法正常使用。（就是为了打倒流氓软件的流氓强制定位）

如果要支持老版本，即 iOS 11 以下系统版本，那么建议在 info.plist 中配置所有的 Key（即使 NSLocationAlwaysUsageDescription 在 iOS 11及以上版本不再使用）：

```objc
NSLocationWhenInUseUsageDescription
NSLocationAlwaysAndWhenInUseUsageDescription
NSLocationAlwaysUsageDescription
```

`NSLocationAlwaysAndWhenInUseUsageDescription`为 iOS 11 中新引入的一个 Key。
参考：[WWDC17: What's New in Location Technologies ?](https://developer.apple.com/videos/play/wwdc2017/713/)
**(这是一个带简体中文字幕的视频，我并没有看！！！)**