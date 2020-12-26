# ReactNative学习笔记

## 桥接原生模块

JS要调用原生的能力，可以是桥接Native Module的方式，其中IOS原生模块需要实现`RCTBridgeModule`协议。

```objective-c
// CalendarManager.h
#import <React/RCTBridgeModule.h>

@interface CalendarManager : NSObject <RCTBridgeModule>
@end
```

其次需要引入`RCT_EXPORT_MODULE()`这个宏，它的作用是声明该原生模块在js段的名称，方便js调用。如果没有明确声明一个模块的名称，这个模块会跟随oc中声明的类名，比如`CalendarManger`，如果类名以`RCT`开头，JS端的模块名会除去`RCT`前缀。

```objective-c
// CalendarManager.m
#import "CalendarManager.h"

@implementation CalendarManager

// To export a module named CalendarManager
RCT_EXPORT_MODULE();

// This would name the module AwesomeCalendarManager instead
// RCT_EXPORT_MODULE(AwesomeCalendarManager);

@end
```

如果要暴露原生方法，要使用`RCT_EXPORT_METHOD();`宏明确告诉JS可以调用的原生方法。

```objective-c
#import "CalendarManager.h"
#import <React/RCTLog.h>

@implementation CalendarManager

RCT_EXPORT_MODULE();

RCT_EXPORT_METHOD(callNativeMethodNoCallback:(NSString *)methodName
                  params:(NSDictionary *)params) {
    NSString *selectorName = [NSString stringWithFormat:@"%@:", methodName];
    SEL selector = NSSelectorFromString(selectorName);
    if ([self respondsToSelector:selector]) {
        GYLOG(Log_RN, @"callNativeMethodNoCallback: %@ %@", methodName, params);
        #pragma clang diagnostic push
        #pragma clang diagnostic ignored "-Warc-performSelector-leaks"
        [self performSelector:selector withObject:params];
        #pragma clang diagnostic pop
    } else {
        GYLOG_WARN(Log_RN, @"unknown methodName: %@", methodName);
        [self callJSMethodWithSubevent:@"RNUnknownBridgeMethodEvent" params:params];
    }
}

@end
```

那么在JS端，可以这样去调用这个方法：
```js
import { NativeModules } from 'react-native';
var CalendarManager = NativeModules.CalendarManager;

WRRCTManager.callNativeMethodNoCallback(methodName, params);
WRRCTManager.callNativeMethodWithCallback(methodName, params);

```

当JS端调用原生方法传递参数的时候，有时候需要进行参数类型的转换，在原生端可以使用`RCTConvert`函数解决这个问题，下例中将JS的date类型转换为了OC的date类型：

```objective-c
RCT_EXPORT_METHOD(addEvent:(NSString *)name location:(NSString *)location date:(nonnull NSNumber *)secondsSinceUnixEpoch)
{
  NSDate *date = [RCTConvert NSDate:secondsSinceUnixEpoch];
}
```

还有一个更重要的功能，就是可以接受一个callback作为参数

```objective-c
RCT_EXPORT_METHOD(findEvents : (RCTResponseSenderBlock)callback)
{
  NSArray *events = ...
  callback(@[[NSNull null], events]);
}
```

在JS端可以调用这个方法并访问到原生传递的参数：

```js
CalendarManager.findEvents((error, events) => {
  if (error) {
    console.error(error);
  } else {
    this.setState({ events: events });
  }
});
```

更重要的是，原生还可以用Promise的形式声明一个方法：

```objective-c
RCT_REMAP_METHOD(findEvents,
                 findEventsWithResolver : (RCTPromiseResolveBlock)resolve
                 rejecter : (RCTPromiseRejectBlock)reject)
{
  NSArray *events = ...
  if (events) {
    resolve(events);
  } else {
    NSError *error = ...
    reject(@"no_events", @"There were no events", error);
  }
}
```

在JS端，可以使用`await async`调用：
```js
const updateEvents = async () => {
  try {
    var events = await CalendarManager.findEvents();

    this.setState({ events });
  } catch (e) {
    console.error(e);
  }
};

updateEvents();
```

RN会将原生方法放进一个顺序串形队列里面，但使用`- (dispatch_queue_t)methodQueue`方法可以改变一个方法处在的线程，比如放在主线程：

```objective-c
- (dispatch_queue_t)methodQueue
{
  return dispatch_get_main_queue();
}
```

如果一个方法是一个长任务，应该要将这个方法放到另外一个线程中执行，否则会长时间阻塞，此时可以创建新的队列：

```objective-c
- (dispatch_queue_t)methodQueue
{
  return dispatch_queue_create("com.facebook.React.AsyncLocalStorageQueue", DISPATCH_QUEUE_SERIAL);
}
```

原生端可以向JSruntime注入常量，js端可以同步访问到这些常量：

```objective-c
- (NSDictionary *) constantsToExport
{
  return @{ @"firstDayOfTheWeek": @"Monday" };
}

// js
console.log(CalendarManager.firstDayOfTheWeek);
```

### 向JS端发送事件

原生端可以发送事件给JS端，实现一个观察者模式，具体是将当前类继承`RCTEventEmitter`，并实现`supportedEvents`以及调用`self sendEventWithName`：

```objective-c
// WRRCTManager.h
#import <React/RCTBridgeModule.h>
#import <React/RCTEventEmitter.h>

#define WRRCTNativeEvent @"WRRCTNativeEvent"

@interface WRRCTManager : RCTEventEmitter <RCTBridgeModule>

@end

// WRRCTManager.m
#import "WRRCTManager.h"

@implementation WRRCTManager

RCT_EXPORT_MODULE();

#pragma mark - Lifecycle

- (NSArray<NSString *> *)supportedEvents
{
  return @[WRRCTNativeEvent];
}`

// 当WRRCTManager在js端监听了一个事件，这个函数会被调用
- (void)startObserving {
  // 注册一个全局的通知中心去处理事件
}

- (void)stopObserving {
    hasListeners = NO;
    // Remove upstream listeners, stop unnecessary background tasks
}

- (void)callJSMethodWithSubevent:(NSString *)subevent params:(NSDictionary *)params {
    if (subevent.length <= 0) {
        return;
    }
    NSMutableDictionary *body = [NSMutableDictionary dictionaryWithDictionary:params];
    body[@"subevent"] = subevent;
    // 调用sendEventWithName向JS端发送事件
    [self sendEventWithName : WRRCTNativeEvent body:body];
}


@end
```

在JS端使用`NativeEventEmitter`去监听原生发生的事件，原生端会调用callback或者promise

```js
import { NativeEventEmitter, NativeModules } from 'react-native';
const WRRCTManager = NativeModules.WRRCTManager;

export const WRRCTNativeEvent = 'WRRCTNativeEvent';
const AllowedEvents = [WRRCTNativeEvent];

class NativeManager {
  constructor () {
    this._nativeEventEmitter = new NativeEventEmitter(WRRCTManager);
  }
  addEventListener (eventName, handler) {
    this._nativeEventEmitter.addEventListener(eventName, handler);
  }
}

// 全局单例
export default new NativeManager;

```

## 桥接原生UI View

原生View都是RCTViewManager的单例子类，要桥接原生view，首先要新建一个viewManager，也就是一个RCTViewManager的子类，然后添加`RCT_EXPORT_MODULE()`宏，再实现`-(UIView *) view`方法。

```objective-c
#import "WRRCTDiscoverActivityCardViewManager.h"
#import "WRRCTDiscoverActivityCardView.h"

@implementation WRRCTDiscoverActivityCardViewManager

RCT_EXPORT_MODULE(WRDiscoverActivityCardView);

- (UIView)view {
  WRRCTDiscoverActivityCardViewManager *view = [[WRRCTDiscoverActivityCardView alloc] init]
}

RCT_EXPORT_VIEW_PROPERTY(cardData, NSDictionary)

@end
```

JS端通过`RCT_EXPORT_VIEW_PROPERTY()`宏可以向原生端传递一些属性，底层会通过RCTConvert对数据进行转换，第二个参数指明了该属性的类型。一般会实现一个wrapper类记录property。

```js
// MapView.js
import PropTypes from 'prop-types';
import React from 'react';
import { requireNativeComponent } from 'react-native';

class MapView extends React.Component {
  render() {
    return <RNTMap {...this.props} />;
  }
}

MapView.propTypes = {
  /**
   * A Boolean value that determines whether the user may use pinch
   * gestures to zoom in and out of the map.
   */
  zoomEnabled: PropTypes.bool
};

var RNTMap = requireNativeComponent('RNTMap', MapView);

module.exports = MapView;
```

## RN手势系统

