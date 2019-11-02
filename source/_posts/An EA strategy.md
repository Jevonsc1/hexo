title: An EA strategy
date: 2016-09-13 15:08:06
tags: "日常"
---

研究智能交易有一定时间了，突发灵感写了个逆势加仓的EA，以此纪念。源代码一字不漏奉上。

<!--more-->

	#property copyright "Copyright © 2016年 ZhengJevons. All rights reserved."
	#property link "http://jevonsc1.github.com"
	#property version   "1.00"
	#property strict

	extern double first_order_count = 0.1; //初始下单量
	extern int first_order_tp = 200; //初始单止盈点数
	extern int gap_point = 100;  //亏损加仓间隔点数
	extern double count_times = 2; //亏损加仓下倍数
	extern int add_times = 5; //亏损加仓次
	extern int long_period=20; //大周期均线
	extern int short_period=10; //小周期均线
	extern double profit_for_close=20; //总获利大于几美金全部平仓
	extern int magic = 1213; //用于区别其他EA的识别码
	datetime buytime=0;
	datetime selltime=0;
	
	
	int OnInit()
	{
		return(INIT_SUCCEEDED);
	}
	
	void OnDeinit(const int reason)
	{
	}
	
	void OnTick()
	{
    /*
    iMA()是移动平均线
    下列变量主要是用来粗略判断金叉，死叉的出现，可以用更好的指标替代。本算法的重点是逆势加仓的手法。
    */
     double long_0=iMA(NULL,0,long_period,0,MODE_SMA,PRICE_CLOSE,0);
     double long_1=iMA(NULL,0,long_period,0,MODE_SMA,PRICE_CLOSE,1);
     double long_2=iMA(NULL,0,long_period,0,MODE_SMA,PRICE_CLOSE,2);
     double short_0=iMA(NULL,0,short_period,0,MODE_SMA,PRICE_CLOSE,0);
     double short_1=iMA(NULL,0,short_period,0,MODE_SMA,PRICE_CLOSE,1);
     double short_2=iMA(NULL,0,short_period,0,MODE_SMA,PRICE_CLOSE,2);
   
     double buyop,buylots;
     //得到多单的数量，并将多单的最后那单的开盘价和手数记录在buyop，buylots中
     int buy_order_count=buy_order_count(buyop,buylots);
     if(buy_order_count==0)
      {
    
     if(short_0>long_0 && short_1<long_1)//若手数为零，同时发生金叉的话买入0.1手（或first_order_count手）
          {
           if(buytime!=Time[0])
            {
              if(buy(first_order_count,0,first_order_tp,Symbol()+"buy",magic)>0)
               {
                 buytime=Time[0];
               }
            }
          }
      }
      else
      {
        if(buy_order_count<add_times && (buyop-Ask)>=gap_point*Point)
        //若手数不为零，且当前价格低于买入价gap_point*Point这么多个点，加倍加仓
         {
           
           buy(flots(buylots*count_times),0,0,Symbol()+"buy"+buy_order_count,magic);
         }
      }
      
      //修改止盈点到平均价
     double avgbuyprice=avgbuyprice();
     if(avgbuyprice>0 && buy_order_count>1)
      {
        buy_change_tp(avgbuyprice);
      }
      
    
     /*下面是空单的做法，与多单的做法一致*/ 
     double sellop,selllots;
     int sell_order_count=sell_order_count(sellop,selllots);
     if(sell_order_count==0)
      {
     //   if(xiao0>da0 && xiao1<da1)
     if(short_0<long_0 && short_1>long_1)

      {
           if(selltime!=Time[0])
            {
              if(sell(first_order_count,0,first_order_tp,Symbol()+"sell",magic)>0)
                {
                  selltime=Time[0];
                }
            }
          }
      }
     else
      {
        if(sell_order_count<add_times && (Bid-sellop)>=gap_point*Point)
         {
           
           sell(flots(selllots*count_times),0,0,Symbol()+"sell"+sell_order_count,magic);
         }
      }
      
      
      
     double avgsellprice=avgsellprice();
     if(avgsellprice>0 && sell_order_count>1)
      {
        sell_change_tp(avgsellprice);
      }
      

      /*无论多单还是空单获利到一个程度就平仓，默认20美元*/
     if(buyprofit()>profit_for_close)
      {
       closebuy();
      }
     if(sellprofit()>profit_for_close)
      {
       closesell();
      }
	 }

	/*计算多单获利情况*/  
	
	double buyprofit()
	{
     double a=0;
     int t=OrdersTotal();
     for(int i=t-1;i>=0;i--)
         {
           if(OrderSelect(i,SELECT_BY_POS,MODE_TRADES)==true)
             {
               if(OrderSymbol()==Symbol() && OrderType()==OP_BUY && OrderMagicNumber()==magic)
                 {
                   a=a+OrderProfit()+OrderCommission()+OrderSwap();
                 }
             }
         }  
         return(a);
    }
  
	/*计算空单获利情况*/  

	double sellprofit()
	{
     double a=0;
     int t=OrdersTotal();
     for(int i=t-1;i>=0;i--)
         {
           if(OrderSelect(i,SELECT_BY_POS,MODE_TRADES)==true)
             {
               if(OrderSymbol()==Symbol() && OrderType()==OP_SELL && OrderMagicNumber()==magic)
                 {
                   a=a+OrderProfit()+OrderCommission()+OrderSwap();
                 }
             }
         }  
    return(a);
    }

	/*平仓函数*/
	  
	void closebuy()
	{ 
    double buyop,buylots;
    while(buy_order_count(buyop,buylots)>0)
     {
        int t=OrdersTotal();
        for(int i=t-1;i>=0;i--)
         {
           if(OrderSelect(i,SELECT_BY_POS,MODE_TRADES)==true)
             {
               if(OrderSymbol()==Symbol() && OrderType()==OP_BUY && OrderMagicNumber()==magic)
                 {
                   OrderClose(OrderTicket(),OrderLots(),OrderClosePrice(),300,Green);
                 }
             }
         }
        Sleep(800);
    	}
     }
     
	void closesell()
	{
    double sellop,selllots;
    while(sell_order_count(sellop,selllots)>0)
     {
        int t=OrdersTotal();
        for(int i=t-1;i>=0;i--)
         {
           if(OrderSelect(i,SELECT_BY_POS,MODE_TRADES)==true)
             {
               if(OrderSymbol()==Symbol() && OrderType()==OP_SELL && OrderMagicNumber()==magic)
                 {
                   OrderClose(OrderTicket(),OrderLots(),OrderClosePrice(),300,Green);
                 }
             }
         }
        Sleep(800);
     } 
    }

	/*修改多单止盈点*/
  
	void buy_change_tp(double tp)
	{
     for(int i=0;i<OrdersTotal();i++)
      {
        if(OrderSelect(i,SELECT_BY_POS,MODE_TRADES)==true)
          {
            if(OrderSymbol()==Symbol() && OrderType()==OP_BUY && OrderMagicNumber()==magic)
              {
                if(NormalizeDouble(OrderTakeProfit(),Digits)!=NormalizeDouble(tp,Digits))
                 {
                   OrderModify(OrderTicket(),OrderOpenPrice(),OrderStopLoss(),tp,0,Green);
                 }
              }
          }
      }
    }

	/*修改空单止盈点*/  

	void sell_change_tp(double tp)
	{
     for(int i=0;i<OrdersTotal();i++)
      {
        if(OrderSelect(i,SELECT_BY_POS,MODE_TRADES)==true)
          {
            if(OrderSymbol()==Symbol() && OrderType()==OP_SELL && OrderMagicNumber()==magic)
              {
                if(NormalizeDouble(OrderTakeProfit(),Digits)!=NormalizeDouble(tp,Digits))
                 {
                   OrderModify(OrderTicket(),OrderOpenPrice(),OrderStopLoss(),tp,0,Green);
                 }
              }
          }
      }
    }

	/*返回多单平均价格，用来修改止盈点*/   

	double avgbuyprice()
	{
    double a=0;
    int count=0;
    double price_sum=0;
    for(int i=0;i<OrdersTotal();i++)
      {
        if(OrderSelect(i,SELECT_BY_POS,MODE_TRADES)==true)
          {
            if(OrderSymbol()==Symbol() && OrderType()==OP_BUY && OrderMagicNumber()==magic)
              {
               price_sum=price_sum+OrderOpenPrice();
               count++;
              }
          }
      }
    if(count>0)
     {
      a=price_sum/count;
     }
    return(a);
    }

	/*返回空单平均价格，用来修改止盈点*/ 
	
	double avgsellprice()
	{
    	double a=0;
    	int count=0;
    	double price_sum=0;
    	for(int i=0;i<OrdersTotal();i++)
    	  {
     	   if(OrderSelect(i,SELECT_BY_POS,MODE_TRADES)==true)
      	    {
       	     if(OrderSymbol()==Symbol() && OrderType()==OP_SELL && OrderMagicNumber()==magic)
              	{
               	price_sum=price_sum+OrderOpenPrice();
               	count++;
            	  }
         	 }
     	 }
    	if(count>0)
     	{
      		a=price_sum/count;
     	}
    	return(a);
    }

	/*格式化下单手数的小数点，以免小数点超出最小位数*/

	double flots(double dlots)
	{
	    double fb=NormalizeDouble(dlots/MarketInfo(Symbol(),MODE_MINLOT),0);
	    return(MarketInfo(Symbol(),MODE_MINLOT)*fb);
	}
  
	/*参数是引用语义的，是希望将多单手数和最新买入的开盘价传递出去，返回的是多单单数*/  

	int buy_order_count(double &op,double &lots) 
	{
     int a=0;
     //  op=0;
     //   lots=0;
     for(int i=0;i<OrdersTotal();i++)
      {
        if(OrderSelect(i,SELECT_BY_POS,MODE_TRADES)==true)
          {
            if(OrderSymbol()==Symbol() && OrderType()==OP_BUY && OrderMagicNumber()==magic)
              {
                a++;
                op=OrderOpenPrice();
                lots=OrderLots();
              }
          }
      }
    return(a);
    }

	/*参数是引用语义的，是希望将空单手数和最新买入的开盘价传递出去，返回的是空单单数*/  
	

	int sell_order_count(double &op,double &lots)
	{
     int a=0;
     //  op=0;
     //  lots=0;
     for(int i=0;i<OrdersTotal();i++)
      {
        if(OrderSelect(i,SELECT_BY_POS,MODE_TRADES)==true)
          {
            if(OrderSymbol()==Symbol() && OrderType()==OP_SELL && OrderMagicNumber()==magic)
              {
                a++;
                op=OrderOpenPrice();
                lots=OrderLots();
              }
          }
      }
    return(a);
    }

	 /*
	 下多单函数
	 返回值是单号
	 中间部分的判断主要是防止复杂网络环境中的重复下单
	 */
 
 	

	int buy(double lots,double sl,double tp,string com,int buymagic)
	{
    int a=0;
    bool found=false;
     for(int i=0;i<OrdersTotal();i++)
      {
        if(OrderSelect(i,SELECT_BY_POS,MODE_TRADES)==true)
          {
            string comment=OrderComment();
            int ma=OrderMagicNumber();
            if(OrderSymbol()==Symbol() && OrderType()==OP_BUY && comment==com && ma==buymagic)
              {
                found=true;
                break;
              }
          }
      }
    if(found==false)
      {
        if(sl!=0 && tp==0)
         {
          a=OrderSend(Symbol(),OP_BUY,lots,Ask,50,Ask-sl*Point,0,com,buymagic,0,White);
         }
        if(sl==0 && tp!=0)
         {
          a=OrderSend(Symbol(),OP_BUY,lots,Ask,50,0,Ask+tp*Point,com,buymagic,0,White);
         }
        if(sl==0 && tp==0)
         {
          a=OrderSend(Symbol(),OP_BUY,lots,Ask,50,0,0,com,buymagic,0,White);
         }
        if(sl!=0 && tp!=0)
         {
          a=OrderSend(Symbol(),OP_BUY,lots,Ask,50,Ask-sl*Point,Ask+tp*Point,com,buymagic,0,White);
         } 
      }
    return(a);
    }

	/*
	下空单函数
	返回值是单号
	中间部分的判断主要是防止复杂网络环境中的重复下单
	*/ 
	
	
	int sell(double lots,double sl,double tp,string com,int sellmagic)
	{
    int a=0;
    bool found=false;
     for(int i=0;i<OrdersTotal();i++)
      {
        if(OrderSelect(i,SELECT_BY_POS,MODE_TRADES)==true)
          {
            string comment=OrderComment();
            int ma=OrderMagicNumber();
            if(OrderSymbol()==Symbol() && OrderType()==OP_SELL && comment==com && ma==sellmagic)
              {
                found=true;
                break;
              }
          }
      }
    if(found==false)
      {
        if(sl==0 && tp!=0)
         {
           a=OrderSend(Symbol(),OP_SELL,lots,Bid,50,0,Bid-tp*Point,com,sellmagic,0,Red);
         }
        if(sl!=0 && tp==0)
         {
           a=OrderSend(Symbol(),OP_SELL,lots,Bid,50,Bid+sl*Point,0,com,sellmagic,0,Red);
         }
        if(sl==0 && tp==0)
         {
           a=OrderSend(Symbol(),OP_SELL,lots,Bid,50,0,0,com,sellmagic,0,Red);
         }
        if(sl!=0 && tp!=0)
         {
           a=OrderSend(Symbol(),OP_SELL,lots,Bid,50,Bid+sl*Point,Bid-tp*Point,com,sellmagic,0,Red);
         }
      }
    return(a);
    }