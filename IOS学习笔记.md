# IOS学习笔记

## Objective-C

`[obj op]`代表通知某个对象去做什么，又叫发送信息或者调用方法。

一个app只有一个UIWindow窗口类，是其他视图的根容器，APP启动后首先会创建一个UIWindow，然后创建根viewController和其的view，将该view添加到UIWindow上，这个view就会显示在屏幕上。

* 类（class）是表示对象类型的结构体
* 对象（object）是包含值和指向其类的指针的结构体
* 实例（instance）是对象的别称
* 消息（message）是对象可以执行的操作，用于通知对象去做什么
* 方法（method）是为了响应消息而运行的代码
* 方法调度（method dispatch）是oc的一种机制
* 接口（interface）是类为对象提供的特性描述
* 实现（implementation）是对接口的实现

### 类

```objective-c
@interface Circle : NSObject {
    @private 
    ShapeColor fillColor;
    ShapeRect bounds;
}

- (void) setFillColor: (ShapeColor) fillColor;
- (void) setBounds: (ShapeRect) bounds;
- (void) darw;

@end
```
在oc中，用`-`声明类的实例方法，`-`后面是函数的返回类型，中缀符`:`表示该方法带参数，而且`:`是方法名的一部分，所以这是两个不同的方法：

```objective-c
- (void) fun1;
- (void) fun1: (int) arg;
```

`@implementation`是一个编译器指令，表示实现某个类，

```objective-c
@implementation Circle

- (void) setFillColor: (ShapeColor) c {
    fillColor = c;
}

- (void) setBounds: (ShapeRect) b {
    bounds = b;
}

- (void) draw {
    NSLog (@"drawing a circle at (%d %d %d %d) in %@", 
        bounds.x, bounds.y,
        bounds.width, bounds.height,
        colorName(fillColor));
}
@end
```

### 实例化对象

在oc中实例化一个类，可以把这个类当做对象，给它发送一个new消息，该类接收到new消息并处理完毕后会得到一个新的对象：

```objective-c
id circle = [Circle new];
```

### 继承

如果子类要改变父类的方法的实现，需要重写（override）继承的写法。

### NSArray

NSArray是一个Cocoa类，用来存储对象的有序列表，有两个限制：
* 只能存储OC对象，不能存储C语言数据类型，比如int、float
* 不能存储nil

可以通过类方法`arrayWithObjects:`创建一个NSArray，在末尾添加nil代表列表结束。

```objective-c
NSArray *array1 = [NSArray arrayWithObjects: @"one", @"two", @"three", nil];
// 使用数组字面量格式创建数组
NSArray *array2 = @[@"one", @"two", @"three"];
// 通过字面量访问数组元素：
id *myObject = array1[1];
// 通过方法访问数组元素
id *myObject2 = [array1 objectAtIndex: 1];
// 通过方法取数组到长度
NSInteger len = [array1 count];
```

切分数组：
```objective-c
NSString *string = @"oop:ack:bork";
NSArray *chunks = [string componentsSeparatedByString: @":"];
// 合并数组
string = [chunks componentsJoinedByString: @":"];
```

可变数组：NSMutableArray，没有字面量创建语法

```objective-c
NSMutableArray *mutableArray = [NSMutableArray arrayWithCapacity: 10];
// 添加对象
[mutableArray addObject: @""];
// 删除对象
[mutableArray removeObjectAtIndex: 0];
```

### 枚举

```objective-c
for (NSString *string in array) {
    NSLog(@"%@", string);
}
```

### NSDictionary

NSDictionary不可变，NSMutableDictionary可变

``objective-c
// 类方法声明
NSDictionary *dict = [NSDictionary dictionaryWithObjectsAndKeys: @"key1", @"val", nil];
// 字面量方式声明
NSDictionary *dict1 = @{@"key1" : @"val1", @"key2" : @"val2"};
// 类方法访问
NSString *val = [dict objectForKey: @"key1"];
// 字面量方式访问
NSString *val1 = dict1[@"key1"];
```

### NSNumber

NSNumber类用来封装基本数据类型

### NSNull

[NSNull null]表示空，相当于nil的作用。

## 内存管理

OC使用引用计数，如果要增加对象的计数值，调用一条retain消息，要减少，则调用一条release消息，要获取当前的计数值可以用retainCount消息。当计数为0的时候，会自动发送dealloc销毁对象。

通过关键字`@autorelease`可以创建一个自动释放池，NSObject上有一个autorelease方法，当给一个对象发送这个消息的时候，会将这个对象添加到自动释放池，当自动释放池被释放的时候，会向池中所有的对象发生release消息。

也可以手动调用这个方法：

```objective-c
- (NSString *) description {
    NSString *desc = [[NSString alloc] initWithFormat: @""];
    return ([desc autorelease]);
}
```

可以通过NSAutoreleasePool手动创造一个新的自动释放池，创建和释放pool之间的发送autorelease消息的对象就会使用这个池子。

```objective-c
NSAutoreleasePool *pool = [NSAutoreleasePool new];
// code here
[pool release];
```

自动释放池是栈的结构，新添加的pool会被push到栈顶

## 概念

### OOP四大原则

* 开放/封闭原则：软件对扩展开放，对修改封闭
* 

## viewController

IOS开发是典型的MVC结构，UIViewController负责管理关联的View，以及与其他的UIViewController通信和协调。一个UIViewController控制管理一个根视图，其他视图都放置在根视图中。

ios中通信的方式：

* model通过Notification和kvo机制与controller层通信
* controller通过view的DataSource属性，设置视图的数据源
* view通过Action Target向viewController报告事件发生
* view通过Delegate机制，向viewController报告事件发生

