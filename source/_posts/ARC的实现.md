title: ARC的实现
date: 2016-05-13 15:05:26
tags: iOS
---

苹果的官方说明中称，ARC 是“由编译器进行内存管理”的，但实际上只有编译器是无法完全胜任的，在此基础上还需要Objective-C 运行时库的协助。也就是说，ARC 由以下工具、库来实现。

* clang（LLVM 编译器）3.0 以上

* objc4 Objective

* -C 运行时库 493.9 以上


<!--more-->

如果按照苹果的官方说明，假设仅由编译器进行ARC 式的内存管理，那么_ _ weak 修饰符也完全可以使用在iOS4 和OS  X  Snow  Leopard 中。但实际上，在编译用于iOS4 和OS  X  Snow Leopard 的应用程序时，并不链接一般使用的库，而是使用libarclite_iphoneos.a 或libarclite_macosx.a 这些旧OS 上用于实现ARC 的库。

不过由于libarclite 的源代码没有公开，以上只是一个推测，但基于安装在iOS4 和OS  X Snow  Leopard 上的遗留框架和运行时库的功能，无论怎样静态链接用于ARC 的库，也不能在对象废弃时将_ _ weak 变量初始化为nil（空弱应用）。

那么下面就让我们彻底地忘记旧OS 的事，基于实现来研究一下ARC 吧，下面将围绕clang 汇编输出和objc4 库（主要是runtime/objc-arr.mm）的源代码进行说明。


__strong 修饰符
============

赋值给附有_ _ strong 修饰符的变量在实际的程序中到底是怎样运行的呢？

	{
  	  id __strong obj = [[NSObject alloc] init];
	}

在编译器选项“-S”的同时运行clang，可取得程序汇编输出。看看汇编输出和objc4 库的源代码就能够知道程序是如何工作的。该源代码实际上可转换为调用以下的函数。为了便于理解，以后的源代码有时也使用模拟源代码。

	/* 编译器的模拟代码 */
	id obj = objc_msgSend(NSObject, @selector(alloc));
	objc_msgSend(obj, @selector(init));
	objc_release(obj);

如原源代码所示，2 次调用objc_msgSend 方法（alloc 方法和init 方法），变量作用域结束时通过objc_release 释放对象。虽然ARC 有效时不能使用release 方法，但由此可知编译器自动插入了release。下面我们来看看使用alloc/new/copy/mutableCopy 以外的方法会是什么情况。

	{
    	id __strong obj = [NSMutableArray array];
	}

虽然调用了我们熟知的NSMutableArray 类的array 类方法，但得到的结果却与之前稍有不同。

	/* 编译器的模拟代码 */
	id obj = objc_msgSend(NSMutableArray,@selector(array));
	objc_retainAutoreleasedReturnValue(obj);
	objc_release(obj);

虽然最开始的array 方法的调用以及最后变量作用域结束时的release 与之前相同，但中间的objc_retainAutoreleasedReturnValue 函数是什么呢？

objc_retainAutoreleasedReturnValue 函数主要用于最优化程序运行。顾名思义，它是用于自己持有（retain）对象的函数，但它持有的对象应为返回注册在autoreleasepool 中对象的方法，或是函数的返回值。像该源代码这样，在调用alloc/new/copy/mutableCopy 以外的方法，即NSMutableArray 类的arra y 类方法等调用之后，由编译器插入该函数。

这种objc_retainAutoreleasedReturnValue 函数是成对的，与之相对的函数是objc_autoreleaseReturnValue。它用于alloc/new/copy/mutableCopy 方法以外的NSMutableArray 类的array 类方法等返回对象的实现上。下面我们看看NSMutableArray 类的array 类通过编译器会进行怎样的转换。

	+ (id) array
	{
 	   return [[NSMutableArray alloc] init];
	}

以下为该源代码的转换，转换后的源代码使用了objc_autoreleaseReturnValue 函数。

	/* 编译器的模拟代码 */
	+ (id) array
	{
    	id obj = objc_msgSend(NSMutableArray, @selector(alloc));
    	objc_msgSend(obj, @selector(init));
    	return objc_autoreleaseReturnValue(obj);
	}

像该源代码这样，返回注册到autoreleasepool 中对象的方法使用了objc_autoreleaseReturnValue 函数返回注册到autoreleasepool 中的对象。但是objc_autoreleaseReturnValue函数同objc_autorelease 函数不同，一般不仅限于注册对象到autoreleasepool 中。

objc_autoreleaseReturnValue 函数会检查使用该函数的方法或函数调用方的执行命令列表，如果方法或函数的调用方在调用了方法或函数后紧接着调用objc_retainAutoreleasedReturnValue(  )函数，那么就不将返回的对象注册到autoreleasepool 中，而是直接传递到方法或函数的调用方。objc_retainAutoreleasedReturnValue 函数与objc_retain 函数不同，它即便不注册到a toreleasepool中而返回对象，也能够正确地获取对象。通过objc_autoreleaseReturnValue 函数和objc_retainAutoreleasedReturnValue 函数的协作，可以不将对象注册到autoreleasepool 中而直接传递，这一过程达到了最优化。

![](http://jevonsc1.github.io/images/strong.png)

__weak 修饰符 
==========

就像前面我们看到的一样，_ _ weak 修饰符提供的功能如同魔法一般。

若附有_ _ weak 修饰符的变量所引用的对象被废弃，则将nil 赋值给该变量。

使用附有_ _ weak 修饰符的变量，即是使用注册到autoreleasepool 中的对象。

这些功能像魔法一样，到底发生了什么，我们一无所知。所以下面我们来看看它们的实现。

	{
   		 id __weak obj1 = obj;
	}
	
假设变量obj 附加_ _ strong 修饰符且对象被赋值。

	/* 编译器的模拟代码 */
	id obj1;
	objc_initWeak(&obj1, obj);
	objc_destroyWeak(&obj1);

通过objc_initWeak 函数初始化附有_ _ weak 修饰符的变量，在变量作用域结束时通过objc_destroyWeak 函数释放该变量。

如以下源代码所示，objc_initWeak 函数将附有_ _ weak 修饰符的变量初始化为0 后，会将赋值的对象作为参数调用objc_storeWeak 函数。

	obj1 = 0;
	objc_storeWeak(&obj1, obj);
	objc_destroyWeak 函数将0 作为参数调用objc_storeWeak 函数。
	objc_storeWeak(&obj1, 0);

即前面的源代码与下列源代码相同。

	/* 编译器的模拟代码 */
	id obj1;
	obj1 = 0;
	objc_storeWeak(&obj1, obj);
	objc_storeWeak(&obj1, 0);

objc_storeWeak 函数把第二参数的赋值对象的地址作为键值，将第一参数的附有_ _ weak 修饰符的变量的地址注册到weak 表中。如果第二参数为0，则把变量的地址从weak 表中删除。weak 表与引用计数表相同，作为散列表被实现。如果使用weak 表，将废弃对象的地址作为键值进行检索，就能高速地获取对应的附有_ _ weak 修饰符的变量的地址。另外，由于一个对象可同时赋值给多个附有_ _ weak 修饰符的变量中，所以对于一个键值，可注册多个变量的地址。

释放对象时，废弃谁都不持有的对象的同时，程序的动作是怎样的呢？下面我们来跟踪观察。对象将通过objc_release 函数释放。

>（1）objc_release

>（2）因为引用计数为0 所以执行dealloc

>（3）_objc_rootDealloc

>（4）object_dispose

>（5）objc_destructInstance

>（6）objc_clear_deallocating


对象被废弃时最后调用的objc_clear_deallocating 函数的动作如下：

>（1）从weak 表中获取废弃对象的地址为键值的记录。

>（2）将包含在记录中的所有附有_ _ weak 修饰符变量的地址，赋值为nil。

>（3）从weak 表中删除该记录。

>（4）从引用计数表中删除废弃对象的地址为键值的记录。

根据以上步骤，前面说的如果附有_ _ weak 修饰符的变量所引用的对象被废弃，则将nil 赋值给该变量这一功能即被实现。由此可知，如果大量使用附有_ _ weak 修饰符的变量，则会消耗相应的CPU 资源。良策是只在需要避免循环引用时使用_ _ weak 修饰符。

使用_ _ weak 修饰符时，以下源代码会引起编译器警告。

	{
	    id __weak obj = [[NSObject alloc] init];
	}

因为该源代码将自己生成并持有的对象赋值给附有_ _ weak 修饰符的变量中，所以自己不能持有该对象，这时会被释放并被废弃，因此会引起编译器警告。

	warning: assigning retained obj to weak variable; obj will be
      released after assignment [-Warc-unsafe-retained-assign]
        id __weak obj = [[NSObject alloc] init];
                  ^     ~~~~~~~~~~~~~~

编译器如何处理该源代码呢？


	/* 编译器的模拟代码 */
	id obj;
	id tmp = objc_msgSend(NSObject, @selector(alloc));
	objc_msgSend(tmp, @selector(init));
	objc_initWeak(&obj, tmp);
	objc_release(tmp);
	objc_destroyWeak(&object);


虽然自己生成并持有的对象通过objc_initWeak 函数被赋值给附有_ _ weak 修饰符的变量中，但编译器判断其没有持有者，故该对象立即通过objc_release 函数被释放和废弃。

这样一来，nil 就会被赋值给引用废弃对象的附有_ _ weak 修饰符的变量中。下面我们通过
NSLog 函数来验证一下。

	{
  	  id __weak obj = [[NSObject alloc] init];
  	  NSLog(@"obj=%@", obj);
	}

以下为该源代码的输出结果，其中用%@ 输出nil。

	obj=(null)

<h4>专栏立即释放对象</h4>

如前所述，以下源代码会引起编译器警告。

	id __weak obj = [[NSObject alloc] init];

这是由于编译器判断生成并持有的对象不能继续持有。附有__unsafe_unretained修饰符的变量又如何呢？

	id __unsafe_unretained obj = [[NSObject alloc] init];

与__weak修饰符完全相同，编译器判断生成并持有的对象不能继续持有，从而发出警告。

	warning: assigning retained object to unsafe_unretained variable;
      obj will be released after assignment [-Warc-unsafe-retained-assign]
    id __unsafe_unretained obj = [[NSObject alloc] init];
                           ^     ~~~~~~~~~~~~~~~~~~~~~~~

该源代码通过编译器转换为以下形式。

	/* 编译器的模拟代码 */
	id obj = objc_msgSend(NSObject, @selector(alloc));
	objc_msgSend(obj, @selector(init));
	objc_release(obj);

objc_release函数立即释放了生成并持有的对象，这样该对象的悬垂指针被赋值给变量obj中。

那么如果最初不赋值变量又会如何呢？下面的源代码在ARC无效时必定会发生内存泄漏。

	[[NSObject alloc] init];

由于源代码不使用返回值的对象，所以编译器发出警告。

	warning: expression result unused [-Wunused-value]
    [[NSObject alloc] init];
    ^~~~~~~~~~~~~~~~~~~~~~~

可像下面这样通过向void型转换来避免发生警告。

	(void)[[NSObject alloc] init];

不管是否转换为void，该源代码都会转换为以下形式

	/* 编译器的模拟代码 */
	id tmp = objc_msgSend(NSObject, @selector(alloc));
	objc_msgSend(tmp, @selector(init));
	objc_release(tmp);

虽然没有指定赋值变量，但与赋值给附有__unsafe_unretained修饰符变量的源代码完全相同。由于不能继续持有生成并持有的对象，所以编译器生成了立即调用objc_release函数的源代码。而由于ARC的处理，这样的源代码也不会造成内存泄漏。

另外，能调用被立即释放的对象的实例方法吗？

	(void)[[[NSObject alloc] init] hash];

该源代码可变为如下形式：

	/* 编译器的模拟代码 */
	id tmp = objc_msgSend(NSObject, @selector(alloc));
	objc_msgSend(tmp, @selector(init));
	objc_msgSend(tmp, @selector(hash));
	objc_release(tmp);

在调用了生成并持有对象的实例方法后，该对象被释放。看来“由编译器进行内存管理”这句话应该是正确的。

这次我们再用附有_ _ weak 修饰符的变量来确认另一功能：使用附有_ _ weak 修饰符的变量，即是使用注册到autoreleasepool 中的对象。

	{
    	id __weak obj1 = obj;
    	NSLog(@"%@", obj1);
	}

该源代码可转换为如下形式：

	/* 编译器的模拟代码 */
	id obj1;
	objc_initWeak(&obj1, obj);
	id tmp = objc_loadWeakRetained(&obj1);
	objc_autorelease(tmp);
	NSLog(@"%@", tmp);
	objc_destroyWeak(&obj1);

与被赋值时相比，在使用附有_ _ weak 修饰符变量的情形下，增加了对objc_loadWeakRetained函数和objc_autorelease 函数的调用。这些函数的动作如下。

* （1）objc_loadWeakRetained 函数取出附有_ _ weak 修饰符变量所引用的对象并retain。

* （2）objc_autorelease 函数将对象注册到autoreleasepool 中。

由此可知，因为附有_ _ weak 修饰符变量所引用的对象像这样被注册到autoreleasepool 中，所以在@autoreleasepool 块结束之前都可以放心使用。但是，如果大量地使用附有_ _ weak 修饰符的变量，注册到autoreleasepool 的对象也会大量地增加，因此在使用附有_ _ weak 修饰符的变量时，最好先暂时赋值给附有_ _ strong 修饰符的变量后再使用。

比如，以下源代码使用了5 次附有_ _ weak 修饰符的变量o。

	{
    	id __weak o = obj;
    	NSLog(@"1 %@", o);
    	NSLog(@"2 %@", o);
    	NSLog(@"3 %@", o);
    	NSLog(@"4 %@", o);
    	NSLog(@"5 %@", o);
	}

相应地，变量o 所赋值的对象也就注册到autoreleasepool 中5 次。

	objc[14481]: ##############
	objc[14481]: AUTORELEASE POOLS for thread 0xad0892c0
	objc[14481]: 6 releases pending.
	objc[14481]: [0x6a85000]  ................  PAGE  (hot) (cold)
	objc[14481]: [0x6a85028]  ################  POOL 0x6a85028
	objc[14481]: [0x6a8502c]         0x6719e40  NSObject
	objc[14481]: [0x6a85030]         0x6719e40  NSObject
	objc[14481]: [0x6a85034]         0x6719e40  NSObject
	objc[14481]:  [0x6a85038]         0x6719e40  NSObject
	objc[14481]: [0x6a8503c]         0x6719e40  NSObject
	objc[14481]: ##############

将附有_ _ weak 修饰符的变量o 赋值给附有_ _ strong 修饰符的变量后再使用可以避免此类问题。

	{
   		id __weak o = obj;
    	id tmp = o;
    	NSLog(@"1 %@", tmp);
    	NSLog(@"2 %@", tmp);
    	NSLog(@"3 %@", tmp);
    	NSLog(@"4 %@", tmp);
    	NSLog(@"5 %@", tmp);
	}
在“tmp = o;”时对象仅登录到autoreleasepool 中1 次。

	objc[14481]: ##############
	objc[14481]: AUTORELEASE POOLS for thread 0xad0892c0
	objc[14481]: 2 releases pending.
	objc[14481]: [0x6a85000]  ................  PAGE  (hot) (cold)
	objc[14481]: [0x6a85028]  ################  POOL 0x6a85028
	objc[14481]: [0x6a8502c]         0x6719e40  NSObject
	objc[14481]: ##############

在iOS4 和OS  X  Snow  Leopard 中是不能使用_ _ weak 修饰符的，而有时在其他环境下也不能使用。实际上存在着不支持_ _ weak 修饰符的类。

例如NSMachPort 类就是不支持_ _ weak 修饰符的类。这些类重写了retain/release 并实现该类独自的引用计数机制。但是赋值以及使用附有_ _ weak 修饰符的变量都必须恰当地使用objc4运行时库中的函数，因此独自实现引用计数机制的类大多不支持_ _ weak 修饰符。

不支持_ _ weak 修饰符的类，其类声明中附加了“_ _ attribute_ _ （（objc_arc_weak_reference_unavailable））”这一属性，同时定义了NS_AUTOMATED_REFCOUNT_WEAK_UNAVAILABLE。

如果将不支持_ _ weak 声明类的对象赋值给附有_ _ weak 修饰符的变量，那么一旦编译器检验出来就会报告编译错误。而且在Cocoa 框架类中，不支持_ _ weak 修饰符的类极为罕见，因此没有必要太过担心。

<h4>专栏allowsWeakReference/retainWeakReference 方法</h4>
实际上还有一种情况也不能使用__weak修饰符。

就是当allowsWeakReference/retainWeakReference实例方法（没有写入NSObject接口说明文档中）返回NO的情况。这些方法的声明如下：

	- (BOOL)allowsWeakReference;
	- (BOOL)retainWeakReference;

在赋值给__weak修饰符的变量时，如果赋值对象的allowsWeakReference方法返回NO，程序将异常终止。

cannot form weak reference to instance (0x753e180) of class MyObject即对于所有allowsWeakReference方法返回NO的类都绝对不能使用__weak修饰符。这样的类必定在其参考说明中有所记述。

另外，在使用__weak修饰符的变量时，当被赋值对象的retainWeakReference方法返回NO的情况下，该变量将使用“nil”。如以下的源代码：

	{
   		id __strong obj = [[NSObjectalloc] init];
    	id __weak o = obj;
    	NSLog(@"1 %@", o);
    	NSLog(@"2 %@", o);
    	NSLog(@"3 %@", o);
    	NSLog(@"4 %@", o);
    	NSLog(@"5 %@", o);
	}

由于最开始生成并持有的对象为附有__strong修饰符变量obj所持有的强引用，所以在该变量作用域结束之前都始终存在。因此如下所示，在变量作用域结束之前，可以持续使用附有__weak修饰符的变量o所引用的对象。

	1 <NSObject: 0x753e180>
	2 <NSObject: 0x753e180>
	3 <NSObject: 0x753e180>
	4 <NSObject: 0x753e180>
	5 <NSObject: 0x753e180>

下面对retainWeakReference方法进行试验。我们做一个MyObject类，让其继承NSObject类并实现retainWeakReference方法。

	@interfaceMyObject : NSObject
	{
 	   NSUInteger count;
	}
	@end
	
	@implementationMyObject
	- (id)init
	{
    	self = [super init];
    	return self;
	}
	
	- (BOOL)retainWeakReference
	{
   	 if (++count > 3)
        	return NO;
    return [super retainWeakReference];
	}
	
	@end

该例中，当retainWeakReference方法被调用4次或4次以上时返回NO。在之前的源代码中，将从NSObject类生成并持有对象的部分更改为MyObject类。

	{
   		id __strong obj = [[MyObject alloc] init];
   		id __weak o = obj;
    	NSLog(@"1 %@", o);
   		NSLog(@"2 %@", o);
    	NSLog(@"3 %@", o);
    	NSLog(@"4 %@", o);
   		NSLog(@"5 %@", o);
	}
以下为执行结果。

	1 <MyObject: 0x753e180>
	2 <MyObject: 0x753e180>
	3 <MyObject: 0x753e180>
	4 (null)
	5 (null)
	
从第4次起，使用附有__weak修饰符的变量o时，由于所引用对象的retainWeakRef-erence方法返回NO，所以无法获取对象。像这样的类也必定在其参考说明中有所记述。

另外，运行时库为了操作__weak修饰符在执行过程中调用allowsWeakReference/retainWeakReference方法，因此从该方法中再次操作运行时库时，其操作内容会永久等待。原本这些方法并没有记入文档，因此应用程序编程人员不可能实现该方法群，但如果因某些原因而不得不实现，那么还是在全部理解的基础上实现比较好。

__autoreleasing 修饰符
========

将对象赋值给附有_ _ autoreleasing 修饰符的变量等同于ARC 无效时调用对象的autorelease方法。我们通过以下源代码来看一下。

	@autoreleasepool {
    	id __autoreleasing obj = [[NSObject alloc] init];
	}

该源代码主要将NSObject 类对象注册到autoreleasepool 中，可作如下变换：

	/* 编译器的模拟代码 */
	id pool = objc_autoreleasePoolPush();
	id obj = objc_msgSend(NSObject, @selector(alloc));
	objc_msgSend(obj, @selector(init));
	objc_autorelease(obj);
	objc_autoreleasePoolPop(pool);

这与苹果的autorelease 实现中的说明（参考1.2.7 节）完全相同。虽然ARC 有效和无效时，其在源代码上的表现有所不同，但autorelease 的功能完全一样。

在alloc/new/copy/mutableCopy 方法群之外的方法中使用注册到autoreleasepool 中的对象会如何呢？下面我们来看看NSMutableArray 类的array 类方法。

	@autoreleasepool {
    	id __autoreleasing obj = [NSMutableArray array];
	}
这与前面的源代码有何不同呢？

	/* 编译器的模拟代码 */
	id pool = objc_autoreleasePoolPush();
	id obj = objc_msgSend(NSMutableArray, @selector(array));
	objc_retainAutoreleasedReturnValue(obj);
	objc_autorelease(obj);
	objc_autoreleasePoolPop(pool);

虽然持有对象的方法从alloc 方法变为objc_retainAutoreleasedReturnValue 函数，但注册autoreleasepool 的方法没有改变，仍是objc_autorelease 函数

<br/>

引用计
=============

实际上，本书为了让读者掌握引用计数式内存管理的思维方式，特地没有介绍引用计数数值本身（只在导入部和Core  Foundation 的转换中稍有说明）。但考虑到有些读者可能极想知道引用计数的数值，因此在这里提供获取引用计数数值的函数。

uintptr_t_objc_rootRetainCount(id obj)如上声明的_ objc_rootRetainCount函数可获取指定对象的引用计数数值。请看以下几个例子。

	{
   	 	id __strong obj = [[NSObject alloc] init];
   		NSLog(@"retain count =%d",_objc_rootRetainCount(obj));
	}

该源代码中，对象仅通过变量obj 的强引用被持有，所以为1。

	retain count = 1

下面使用_ _ weak 修饰符。

	{
    	id __strong obj = [[NSObject alloc] init];
    	id __weak o = obj;
    	NSLog(@"retain count = %d", _objc_rootRetainCount(obj));
	}

由于弱引用并不持有对象，所以赋值给附有_ _ weak 修饰符的变量中也必定不会改变引用计数数值。

	retain count = 1

结果同预想一样。那么通过_ _ autoreleasing 修饰符向autoreleasepool 注册又会如何呢？

	@autoreleasepool {
    	id __strong obj = [[NSObject alloc] init];
    	id __autoreleasing o = obj;
    	NSLog(@"retain count = %d", _objc_rootRetainCount(obj));
	}

结果如下：

	retain count = 2

对象被附有_ _ strong 修饰符变量的强引用所持有，且被注册到autoreleasepool 中，所以为2。

以下确认@autoreleasepool 块结束时释放已注册的对象。

	{
   		 id __strong obj = [[NSObject alloc] init];
    	@autoreleasepool {
       		 id __autoreleasing o = obj;
       		 NSLog(@"retain count = %d in @autoreleasepool", _objc_rootRetainCount(obj));
    	}
   		 NSLog(@"retain count = %d", _objc_rootRetainCount(obj));
	}

在@autoreleasepool 块之后也显示引用计数数值。
retain count = 2 in @autoreleasepool
retain count = 1

如我们预期的一样，对象被释放。

以下在通过附有_ _ weak 修饰符的变量使用对象时，基于显示autoreleasepool 状态的_objc_autoreleasePoolPrint 函数来观察注册到autoreleasepool 中的引用对象。

	@autoreleasepool {
   		 id __strong obj = [[NSObject alloc] init];
    	_objc_autoreleasePoolPrint();
    	id __weak o = obj;
    	NSLog(@"before using __weak: retain count = %d", _objc_rootRetainCount(obj));
    	NSLog(@"class = %@", [o class]);
    	NSLog(@"after using __weak: retain count = %d", _objc_rootRetainCount(obj));
    	_objc_autoreleasePoolPrint();
	}

结果如下：

	objc[14481]: ##############
	objc[14481]: AUTORELEASE POOLS for thread 0xad0892c0
	objc[14481]: 1 releases pending.
	objc[14481]: [0x6a85000]  ................  PAGE  (hot) (cold)
	objc[14481]: [0x6a85028]  ################  POOL 0x6a85028
	objc[14481]: ##############
	before using __weak: retain count = 1
	class = NSObject
	after using __weak: retain count = 2
	objc[14481]: ##############
	objc[14481]: AUTORELEASE P OOLS for thread 0xad0892c0
	objc[14481]: 2 releases pending.
	objc[14481]: [0x6a85000]  ................  PAGE  (hot) (cold)
	objc[14481]: [0x6a85028]  ################  POOL 0x6a85028
	objc[14481]: [0x6a8502c]         0x6719e40  NSObject
	objc[14481]: ##############

通过以上过程我们可以看出，不使用_ _ autoreleasing 修饰符，仅使用附有_ _ weak 声明的变量也能将引用对象注册到了autoreleasepool 中。

虽然以上这些例子均使用了_objc_rootRetainCount 函数，但实际上并不能够完全信任该函数取得的数值。对于已释放的对象以及不正确的对象地址，有时也返回“1”。另外，在多线程中使用对象的引用计数数值，因为存有竞态条件的问题，所以取得的数值不一定完全可信。

虽然在调试中_objc_rootRetainCount 函数很有用，但最好在了解其所具有的问题的基础上来使用。