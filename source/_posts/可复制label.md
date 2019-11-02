title: 可复制label
date: 2016-06-28 16:26:19
tags: iOS
---
在iOS8 之后, 我们发现UILabel不在为我们提供长按弹出复制等操作了, 我们来继承UILabel自己写一个带复制功能的按钮

<!--more-->

	//
	//  XLabel.m
	//  Zstoday
	//
	//  Created by ZhengJevons on 16/6/28.
	//  Copyright © 2016年 ZhengJevons. All rights reserved.
	//

	#import "XLabel.h"

	@implementation XLabel

	-(BOOL)canBecomeFirstResponder {

    	return YES;
	}

	// 可以响应的方法
	-(BOOL)canPerformAction:(SEL)action withSender:(id)sender {

    	return (action == @selector(copy:));
	}

	//针对于响应方法的实现
	-(void)copy:(id)sender {

    	UIPasteboard *pboard = [UIPasteboard generalPasteboard];
    	pboard.string = self.text;
	}

	//UILabel默认是不接收事件的，我们需要自己添加touch事件
	-(void)attachTapHandler {

    	self.userInteractionEnabled = YES;
    	UILongPressGestureRecognizer *touch = [[UILongPressGestureRecognizer alloc] initWithTarget:self action:@selector(handleTap:)];
    	[self addGestureRecognizer:touch];
	}

	//绑定事件
	- (id)initWithFrame:(CGRect)frame {
    	self = [super initWithFrame:frame];
    	if (self) {

        	[self attachTapHandler];
    	}
    	return self;
	}

	-(void)awakeFromNib {

    	[super awakeFromNib];
    	[self attachTapHandler];
	}

	-(void)handleTap:(UIGestureRecognizer*) recognizer {

    	[self becomeFirstResponder];
    	UIMenuItem *copyLink = [[UIMenuItem alloc] initWithTitle:@"复制"
                                                      action:@selector(copy:)];
    	[[UIMenuController sharedMenuController] setMenuItems:[NSArray arrayWithObjects:copyLink, nil]];
    	[[UIMenuController sharedMenuController] setTargetRect:self.frame inView:self.superview];
	    [[UIMenuController sharedMenuController] setMenuVisible:YES animated: YES];
	}

@end