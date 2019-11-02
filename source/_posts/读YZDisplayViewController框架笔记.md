title: 读YZDisplayViewController框架笔记
date: 2016-05-25 15:43:52
tags: iOS
---
前言
---
现在有太多应用有类似腾讯视频多页滑动的需求，类似功能的代码我看了不下10份，自己也亲自码了几个版本出来，直到我看完啊铮的“YZDisplayViewController”之后，才真正感到满意。
![](http://upload-images.jianshu.io/upload_images/304825-4e3ab3a122dc3b25.gif?imageMogr2/auto-orient/strip)

<!---more--->

兼容多种自定义
---
我不知道在这里用兼容这两个会不会造成一些误解，但一想到这个框架万金油式地使用方式也只能用这个词了。
在滑动页面时，我们的标题少不了会有动画，甚至我们的滑动页面还会有其它的控件存在，这导致了我一定要能改框架中的成员变量。很多其它的框架会使用宏的方式把可自定义的部分暴露出来，这样的可维护性虽然时有提高，但是一个应用有多种滑动页风格时，就会有比较大的麻烦。

而啊铮的这个框架采用的是block的形式

整个框架的自定义分为几个部分
* 内容部分，也就是主体部分
* 标题部分
* 下标部分
* 字体缩放部分
* 颜色渐变部分
* 遮盖部分


    /**************************************内容************************************/
	/**
    内容是否需要全屏展示
    YES :  全屏：内容占据整个屏幕，会有穿透导航栏效果，需要手动设置tableView额外滚动区域
    NO  :  内容从标题下展示
	 */
	@property (nonatomic, assign) BOOL isfullScreen;

	/**
    根据角标，选中对应的控制器
	 */
	@property (nonatomic, assign) NSInteger selectIndex;

	/**
    如果_isfullScreen = Yes，这个方法就不好使。
 
    设置整体内容的frame,包含（标题滚动视图和内容滚动视图）
    */
    - (void)setUpContentViewFrame:(void(^)(UIView *contentView))contentBlock;

	/**************************************内容************************************/
	
标题的成员变量采用block赋值	
	
	/**
	 标题滚动视图背景颜色
	 */
	@property (nonatomic, strong) UIColor *titleScrollViewColor;


	/**
	 标题高度
	 */
	@property (nonatomic, assign) CGFloat titleHeight;

	/**
    正常标题颜色
    */
	@property (nonatomic, strong) UIColor *norColor;

	/**
    选中标题颜色
    */
	@property (nonatomic, strong) UIColor *selColor;

	/**
    标题字体
	*/
	@property (nonatomic, strong) UIFont *titleFont;

	// 一次性设置所有标题属性
	- (void)setUpTitleEffect:(void(^)(UIColor **titleScrollViewColor,UIColor **norColor,UIColor **selColor,UIFont **titleFont,CGFloat *titleHeight))titleEffectBlock;	

其它部分同理


内存管理
---
虽然很多的框架都会处理内存的问题，就是当多个页面快速滑动时，机器是如何处理释放和加载页面的呢。啊铮采用的是UIColloctionView加上kvo的方式。

UIColloctionView自带复用机制，当我们要显示哪个页面时它就会从自控制器数组加载相应的数据进行展示，所以我们释放的是页面而不是数据本身。

而用kvo的方式进行响应下拉刷新，可以解决delegate所带来结构上的混乱

细节
---
最感人的细节当然是刷新功能，只要对childViewControllers这个数组重新赋值就可以轻松实现刷新，当然这也是	UIColloctionView的功劳

	- (void)refreshDisplay
	{
		// 清空之前所有标题
		[self.titleLabels makeObjectsPerformSelector:@selector(removeFromSuperview)];
		[self.titleLabels removeAllObjects];
    
		 // 刷新表格
		[self.contentScrollView reloadData];
    
		// 重新设置标题
		[self setUpTitleWidth];
    
    	[self setUpAllTitle];
    
	}