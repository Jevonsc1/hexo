title: 从零实现oop
date: 2017-03-17 09:22:24
tags: iOS
---
![](http://jevonsc1.github.io/images/ow_baolei.jpeg)
<!--more-->
前言
---
为了了解面对对象的特性，没有比用c语言这样一门语言从零开始实现一次更彻底。首先很感激生命中出现了一位臧老师这样的导师带着我们折腾。

面向对象特性
---
用过面向对象语言的人都知道，面向对象的一些基本组成成分

例如：

* 类
* 实例
* 状态（属性集）
    性质有下：
    1. 独特性
    2. 私密性
    3. 可变性
* 行为（方法集）
    性质有下：
    1. 同一类具备同一行为
    2. 查询自己的状态
    3. 改变自己的状态
    4. 触发其他实例的行为
* 关系
    1. 类的关系
    2. 实例的关系
    3. 行为的关系

归纳起来就3个重要的特性： 封装性，继承性，多态性

封装性的性质：
* 行为与状态结合形成一个整体
* 隐藏细节
* 公开接口

继承性：
* 拓展现有封装
* 区别“基于对象”和“面向对象”

多态性：
* 相同行为子类多种实现
* 通过父类抽象访问行为
* 不同子类给以不同响应


 初步实现
---
####结构体表示类

* 结构体定义－>结构体变量
* 类－>实例
* 抽象－>实体

```
typedef struct ClassA {
    int property1;
    float property2;
} ClassA;

ClassA instanceA = {
    .property1 = 1,
    .property2 = 1.5f
};
```
除了用结构体字段来表示状态还可以用kv存储，连续地址偏移量的方式，另外用结构体存状态不能在运行时自省个，也不利于隐藏细节


####函数聚合表示行为

方法有3个要素，执行者，方法名，参数，不同的面向对象语言有自己存放方法的方法，c++语言就是散列存放的，javascript是存放在实例当中，oc、ruby存放在类当中

散列存放方式
* 实现简单
* 理解简单
* 无法实现封装
* 无法重载
```
void police_arrest(Police_Officer *police, User *somebody);
void police_call_partner(Police_Officer *police, Police_Officer *another);
int police_level(Police_Officer *police);

void CGContextBeginPath(CGContextRef cg_nullable c);
void CGContextMoveToPoint(CGContextRef cg_nullable c,
    CGFloat x, CGFloat y);
void CGContextAddRects(CGContextRef cg_nullable c,
    const CGRect * __nullable rects, size_t count);
CGPoint CGContextConvertPointToUserSpace(CGContextRef cg_nullable c,
    CGPoint point);
```

存放于实例的方式
* 实例容易替换方法
* 理解简单
* 查询方法简单
* 父类方法不易找到
* 对象占用过多内存

```
typedef void Arrest_Type(struct Police_Man *self,
                         struct Thief *thief);

typedef void Call_Partner_Type(struct Police_Man *self,
                               struct Police_Man *partner);


typedef struct Police_Man {
    Arrest_Type *arrest;
    Call_Partner_Type *call_partner;
    int police_id;
} Police_Man;                             
```

存放于类
* 内存更节约
* 便于实现继承链
* 对象和类的模型更复杂
```
typedef void Arrest_Type(struct Police_Man *self,
                         struct Thief *thief);

typedef void Call_Partner_Type(struct Police_Man *self,
                               struct Police_Man *partner);

typedef struct Police_Man_MTable {
    Arrest_Type *arrest;
    Call_Partner_Type *call_partner;
} Police_Man_MTable;                     

typedef struct Police_Man {
    Police_Man_MTable *mtable;
    int police_id;
} Police_Man;

Police_Man_MTable global_police_man_mtable;          
```

采用方案
---

* 属性：结构体表示字段
* 类和实例：结构体定义、结构体变量
* 方法存放：存放在类上

####实现封装性

```
typedef void Arrest_Type(struct Police_Man *self,
                         struct Thief *thief);

typedef void Call_Partner_Type(struct Police_Man *self,
                               struct Police_Man *partner);

typedef struct Police_Man_MTable {
    Arrest_Type *arrest;
    Call_Partner_Type *call_partner;
} Police_Man_MTable;

typedef struct Police_Man {
    Police_Man_MTable *mtable;
    int police_id;
} Police_Man;

Police_Man_MTable global_police_man_mtable;
```

```
police_man->mtable->arrest(police_man, thief);
int id = police_man->police_id;
```

```
void arrest_imp(struct Police_Man *self,
                struct Thief *thief) {
    printf("arrest a thief!\n");
}

void call_parnter_imp(struct Police_Man *self,
                      struct Police_Man *partner) {
    printf("call another police man!\n");
}

Police_Man_MTable global_police_man_mtable = {
    .arrest = arrest_imp,
    .call_partner = call_parnter_imp
};

Police_Man *new_police_man() {
    Police_Man *new_obj = (Police_Man *)malloc(sizeof(Police_Man));
    new_obj->mtable = &global_police_man_mtable;
    static int auto_increase_id = 0;
    new_obj->police_id = ++auto_increase_id;
    return new_obj;
}
```

隐藏细节
```
typedef struct Thief {
    struct Thief_MTable *mtable;
    float money;
} Thief;
```
```
typedef struct Thief_Private_Data {
    int tools_count;
} Thief_Private_Data;

typedef struct Thief_Private {
    struct Thief_MTable *mtable;
    float money;
    Thief_Private_Data *private;
} Thief_Private;
```

```
typedef void Dealloc_Type(struct Thief *self);

typedef int Tools_Count_Type(struct Thief *self);

typedef struct Thief_MTable {
    Dealloc_Type *dealloc;
    Tools_Count_Type *tools_count;
} Thief_MTable;
```
```
void thief_dealloc(struct Thief_Private *self) {
    free(self->private);
    free(self);
}

int thief_tools_count(struct Thief_Private *self) {
    return self->private->tools_count;
}

Thief_MTable global_thief_mtable = {
    .dealloc = (Dealloc_Type *)thief_dealloc,
    .tools_count = (Tools_Count_Type *)thief_tools_count
};
```
```
Thief *new_thief() {
    Thief_Private *new_obj = (Thief_Private *)malloc(sizeof(Thief_Private));
    new_obj->mtable = &global_thief_mtable;
    new_obj->private = (Thief_Private_Data *)malloc(sizeof(Thief_Private_Data));
    return (Thief *)new_obj;
}

```

####实现继承性

car是父类，扩写一个子类taxi
```
struct Car;

typedef void Car_Dealloc_Type(struct Car *self);

typedef void Drive_Type(struct Car *self, int meter);

typedef int DriveTimes_Type(struct Car *self);

typedef const char *Color_Type(struct Car *self);

typedef struct Car_MTable {
    Car_Dealloc_Type *dealloc;
    Drive_Type *drive;
    DriveTimes_Type *drive_times;
    Color_Type *color;
} Car_MTable;

Car_MTable global_car_mtable;

typedef struct Car {
    Car_MTable *mtable;
    int total_meters;
    void *private;
} Car;
```
属性扩写
```
typedef struct Car {
    Car_MTable *mtable;
    int total_meters;
    void *private;
} Car;

 typedef struct Taxi {
    Car super;
    float change;
    void *private;
} Taxi;

Taxi *taxi = (Taxi *)malloc(sizeof(Taxi));
Car *car = (Car *)taxi;
```
方法扩写
```
typedef struct Car_MTable {
    Car_Dealloc_Type *dealloc;
    Drive_Type *drive;
    DriveTimes_Type *drive_times;
    Color_Type *color;
} Car_MTable;

Car_MTable global_car_mtable;

typedef struct Taxi_MTable {
    Car_MTable super;
    Taxi_Init_Type *init;
    Available_Type *available;
    Pick_Up_Type *pick_up;
    Set_Off_Type *set_off;
} Taxi_MTable;

Taxi_MTable global_taxi_mtable;

```

####实现多态性
```
void taxi_drive(Taxi *self, int meters) {
    global_car_mtable.drive((Car *)self, meters);
    self->change += meters;
}
```

```
Taxi *new_taxi() {
    Taxi *new_obj = (Taxi *)malloc(sizeof(Taxi));
    ((Car *)new_obj)->mtable = (Car_MTable *)&global_taxi_mtable;
    global_taxi_mtable.super.drive = (Drive_Type *)taxi_drive;
    return new_obj;
}
```
```
void test3() {
    Taxi *taxi = new_taxi();
    Car *car = new_car();
    ((Car *)taxi)->mtable->drive((Car *)taxi, 15);
    car->mtable->drive(car, 17);
}
```