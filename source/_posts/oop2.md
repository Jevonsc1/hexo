---
title:OOP的补充
date:2017-03-30 15:58:14
tags:-iOS
---

补充两种实现oop的方法
-

1. 使用方法放在结构体变量的方式实现OOP

```c

typedef struct Object {
   float  change;
   void *private;
   Taxi_Init_Type *init;
   Available_Type *available;
   Set_off_Type *setoff;
} Object;

Object *base_init(struct Object* self){
    self->change = xxxxx;
    self->init = xxxxxx;
    return self;
}

typedef struct Car{
    Object super;
    Car_Init_Type *init;
}Car;

Car *car_init(struct Car *self){
    base_init((Object *)self);
    return self;
}

Car *get_global_car(){
    static Car *car = NULL;
    if(NULL == car){
        car = car_init(malloc(typeof(Car)));
    }
    return car;
}

void drive(Taxi* self ,int meter){
    car->drive(self,meter);
    xxxxxx
}

    
```

2. 使用KV存储的方式实现属性

```c

struct KVNode;
typedef struct KVNode {
    struct *next;
    const char *key;
    void *value;
    int lenth;
} KVNode;

typedef struct Object{
    Object_Mtable *mtable;
    KVNode *head;
};

void set_kv(Object *o, const char *name,void *value, int length){
    KVNode *node = o -> head;
    while(node != NULL){
        if(strcmp(name,node->key) == 0){
            memcpy(node->value,value,length);
            return;
        }
        if(node -> next == NULL){
            KVNode *new_node = malloc(typeof(KVNode));
            new_code->name = malloc(strlen(name)+1);
            memcpy(new_node->name,name,strlen(name)+1);
            new_code->value = malloc(length);
            memcpy(new_code->value.value,length);
            new_code->length = length;
            node->next = new_code;
            return;
        }
        node = node->next;
    }
}
	
	
```