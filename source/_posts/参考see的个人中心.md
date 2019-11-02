title: 参考see的个人中心
date: 2016-05-26 15:37:36
tags: iOS
---

前言
---
至从一位好友去了“see”工作之后，我就一直对see这个软件持有比较大的关注，对于它的一些UI效果也会忍不住去尝试实现，这次就尝试实现个人中心。

![](/images/see.gif)

<!---more--->

结构分析
---

![](/images/see1.png)

通过基本的交互可以将结构分成4个部分

* 导航条
* 头部
* tabbar部分
* 列表内容部分

需求分析
---

* 整个页面都是可以被拖动的，包括头部
* 点击tabbar部分实现切换
* tabbar可以跟随滚动，但不超出导航条
* 导航条实现透明度变化效果

整个项目的结构基本是从“tabbar可以跟随滚动，但不超出导航条”这个需求出发去构建的。根据这个需求，我们可以想到当下面的tableview滑动时，头部的高度不断变小直到0，这个根据约束的条件tabbar就可以停在导航条下面了。

为tableview设置额外滚动区域，为了可以穿透头部

    - (void)viewDidLoad {
    [super viewDidLoad];
    
    NSLog(@"viewDidLoad");
    
    _lastOffsetY = -(YZHeadViewH + YZTabBarH);
    
    // 设置顶部额外滚动区域
    self.tableView.contentInset = UIEdgeInsetsMake(YZHeadViewH + YZTabBarH , 0, 0, 0);
    
    YZTableView *tableView = (YZTableView *)self.tableView;
    tableView.tabBar = _tabBar;
    

    
	}

设置多个tableview，传递头部视图的高度，为了当tableview滑动时可以改变

    - (void)setUpChildControlller
	{
    	
    
    	for (YZPersonTableViewController *personChildVc in self.childViewControllers) {
        
        // 传递tabBar，用来判断点击了哪个按钮
        personChildVc.tabBar = _tabBar;
        
        // 传递高度约束，用来移动头部视图
        personChildVc.headHCons = _headViewCons;
        
        // 传递标题控件，设置文字透明
        personChildVc.titleLabel = _titleLabel;
        
    	}
	}
	
tableview滑动的代理方法

	- (void)scrollViewDidScroll:(UIScrollView *)scrollView
	{
		 // 获取当前偏移量
    	CGFloat offsetY = scrollView.contentOffset.y;
    
    	// 获取偏移量差值
    	CGFloat delta = offsetY - _lastOffsetY;
    
    	CGFloat headH = YZHeadViewH - delta;
    
    	if (headH < YZHeadViewMinH) {
        	headH = YZHeadViewMinH;
    	}
    
    	_headHCons.constant = headH;
    
    	// 计算透明度，刚好拖动200 - 64 136，透明度为1
    
    	CGFloat alpha = delta / (YZHeadViewH - YZHeadViewMinH);
    
    	// 获取透明颜色
    	UIColor *alphaColor = [UIColor colorWithWhite:0 alpha:alpha];
    	[_titleLabel setTextColor:alphaColor];
    
    	// 设置导航条背景图片
    	[self.navigationController.navigationBar setBackgroundImage:[UIImage imageWithColor:[UIColor colorWithWhite:1 alpha:alpha]] forBarMetrics:UIBarMetricsDefault];
    
	}	
	
更多细节
---
当头部的UserinteractionEnabled 为 no时，响应者链条就会把响应传递到tableview处，这样头部也可以实现滑动。

为了让这个框架具有良好的扩展性，即我不需要手动告诉框架有多少个tableview，它会根据childcontroller的数量自动创建tabbar中的按钮数，并且能准确实现监听。这时用touchbegin监听点击，通知方法实现传递事件就可以有很良好扩展性。

	- (void)touchesBegan:(NSSet *)touches withEvent:(UIEvent *)event
	{
    	UITouch *touch = [touches anyObject];
    	CGPoint curP = [touch locationInView:self];
    
    
    	for (UIView *tabBarChildView in _tabBar.subviews) {
        
        	CGPoint childP = [self convertPoint:curP toView:tabBarChildView];
        
        	if ([tabBarChildView pointInside:childP withEvent:event]) {
            	// 点击了按钮
            
            	// 通知个人控制器切换内容视图
            	[[NSNotificationCenter defaultCenter] postNotificationName:YZClickBtnNote object:nil userInfo:@{YZClickBtnObjcKey: tabBarChildView}];
            
            	// 处理完事件，结束当前事件处理
            	return;
            
        	}
    	}
    
    	// 如果没有处理事件，就调用系统自带的处理方式
    	[super touchesBegan:touches withEvent:event];

	}


在代码结构中，具体实现的类可以在viewdidload方法中为那些图片啊，子控制器初始化什么，而在父类willdidappear方法中就可以对这些可以赋值的变量进行处理，这样代码的风格就可以得到良好维护。

	- (void)viewWillAppear:(BOOL)animated
	{
    	[super viewWillAppear:animated];
    
    	// 即将显示的时候做一次初始化操作
    	static dispatch_once_t onceToken;
    
    	dispatch_once(&onceToken, ^{

        	[self setUpNav];
        
        	// 设置子控制器
        	[self setUpChildControlller];
        
        	// 设置tabBar
        	[self setUpTabBar];
    	});
    
	}
	
存在无法解决的问题
---		

无论在see中还是在我这个框架中都有一个问题，当a列表被滑动了100，切换到b列表滑动200，在切换到a列表发现无端端跟着也被滑动了200。这是这种UI结构所带来无法解决的问题。