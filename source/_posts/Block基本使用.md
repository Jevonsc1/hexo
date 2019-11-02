title: Block基本使用
date: 2016-05-17 16:19:27
tags:
---
Block的声明和线程安全
---

Block属性的声明，首先需要用copy修饰符，因为只有copy后的Block才会在堆中，栈中的Block的生命周期是和栈绑定的。
<!---more--->
另一个需要注意的问题是关于线程安全，在声明Block属性时需要确认“在调用Block时另一个线程有没有可能去修改Block？”这个问题，如果确定不会有这种情况发生的话，那么Block属性声明可以用nonatomic。如果不肯定的话（通常情况是这样的），那么你首先需要声明Block属性为atomic，也就是先保证变量的原子性（Objective-C并没有强制规定指针读写的原子性，C#有）。

比如这样一个Block类型：

	typedef void (^MyBlockType)(int);
属性声明：

	@property (copy) MyBlockType myBlock;
这里ARC和非ARC声明都是一样的，当然注意在非ARC下要release Block。

 

但是，有了atomic来保证基本的原子性还是没有达到线程安全的，接着在调用时需要把Block先赋值给本地变量，以防止Block突然改变。因为如果不这样的话，即便是先判断了Block属性不为空，在调用之前，一旦另一个线程把Block属性设空了，程序就会crash，如下代码：

	if (self.myBlock)
	{
 	   //此时，走到这里，self.myBlock可能被另一个线程改为空，造成crash
	    //注意：atomic只会确保myBlock的原子性，这种操作本身还是非线程安全的
	    self.myBlock(123);
	}
 

所以正确的代码是（ARC）：

	MyBlockType block = self.myBlock;
	//block现在是本地不可变的
	if (block)
	{
	    block(123);
	}
在非ARC下则需要手动retain一下，否则如果属性被置空，本地变量就成了野指针了，如下代码：

//非ARC

	MyBlockType block = [self.myBlock retain];
	if (block)
	{
	  	  block(123);
	}
	[block release];

<br/>
 
 循环引用问题
 ---

循环引用是另一个使用Block时常见的问题。

在ARC下，由于_ _block抓取的变量一样会被Block retain，所以必须用弱引用才可以解决循环引用问题，iOS 5之后可以直接使用  _ _weak，之前则只能使用 _ _unsafe_unretained了， _ _unsafe_unretained缺点是指针释放后自己不会置空。示例代码：

	//iOS 5之前可以用__unsafe_unretained
	//__unsafe_unretained typeof(self) weakSelf = self;
	__weak typeof(self) weakSelf = self;
	self.myBlock = ^(int paramInt)
	{
	    //使用weakSelf访问self成员
	    [weakSelf anotherFunc];
	};
 

在非ARC下，显然无法使用弱引用，这里就可以直接使用__block来修饰变量，它不会被Block所retain的，参考代码：

//非ARC

	__block typeof(self) weakSelf = self;
	self.myBlock = ^(int paramInt)
	{
    //使用weakSelf访问self成员
	    [weakSelf anotherFunc];
	};