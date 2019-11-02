title: Arduino-7段LED
date: 2016-05-12 09:00:09
tags: "Arduino"
---
![](http://images.cnitblog.com/blog/90297/201304/03113351-26486b4b85ba49fea0569e2c073b690c.jpg)

<!--more-->

我使用的是7段共阳极LED显示器给出LED的针脚说明



![](http://images.cnitblog.com/blog/90297/201304/09191911-be88cbdcdf50485a89ee8438dc2b542a.png)

Arduino的输出端口为3~10下面给出Arduino的输出端口对应的LED显示

![](http://images.cnitblog.com/blog/90297/201304/09192220-8abd33f5b4c64a9eba5ca37208301aa0.png)

实现思路为 将Arduino的3~10端口电位置为HIGH，通过调整3~9的电位值，来控制7段LED灯的亮和灭。下面给出Arduino的输出端口对应的LED显示的数字

![](http://images.cnitblog.com/blog/90297/201304/09192737-dd85a56b08b242d693ba07ac5b040f07.png)

通过设置上面对应的输出端口的电位值为LOW，就可以显示对应数字
   
   
    int i=0;
    int j=0;
    int k=0;

	void setup()
	{
	  for(i=3;i<=10;i++)
	  {
	    pinMode(i,OUTPUT);
	  }
	   for(i=3;i<=10;i++)
	    {
	       digitalWrite(i,HIGH);
	    }
	}
	
	void loop()
	{
	 	int num[10][7]={
	 	{3,4,6,7,8,9},
	 	{8,9},
    	{3,5,6,7,8},
        {3,5,7,8,9},
        {4,5,8,9},
        {3,4,5,7,9},
        {3,4,5,6,7,9},
        {3,8,9},
        {3,4,5,6,7,8,9},
        {3,4,5,7,8,9}
     };
    	 for(i=0;i<10;i++)
     	{
     		for(j=0;j<7;j++)
     		{
     			digitalWrite(num[i][j],LOW);
     		}
     		delay(500);
     		for(k=3;k<=9;k++)
     		{
              digitalWrite(k,HIGH);
            }
            delay(500);
        }
      }