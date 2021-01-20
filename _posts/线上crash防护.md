layout: post
title: "线上crash防护"
date: 2021-01-20 20:20:11 -0000
categories: Crash防护 随笔

### 主要目的

- 既可以在debug 下发现crash，又可以在线上预防crash，降低线上的crash 率，提高用户体验，并且crash发生后可以把crash堆栈上传到后台。<br />

### 已经支持的：

- collection类，unrecognized selector，子线程刷新UI检测<br />

### 具体方式：

- debug 下 直接使用AlterView 弹框，显示crash 信息，log 里面打印crash线程。不去用 assert 可以不用阻断UI 流程。release下直接把所有堆栈上传到后台。
- 用弹框，不用assert，是因为如果崩了，主流程就走不下去，测试人员或者下级模块开发人员就不能继续了

### 预防 crash 日志优先级：

- 优先级比线上crash 优先级更高。需要优先处理，因为预防crash 可能会是业务异常。<br />

### 获取异常的方式：

- 通过 try catch finally 来获取crash reason



```objectivec
@try {    
} @catch (NSException *exception) {
} @finally {
}
```

<br />
组件地址 [https://code.registry.wgine.com/tuyaIOS/TYCrashProtection.git](https://code.registry.wgine.com/tuyaIOS/TYCrashProtection.git)


### Collection类
首先看下下面的代码：

```objectivec
 id mybeAnArray = /**/;
 if ( [mybeAnArray class]==[NSArray class] ){
    //will never be hit  
 }
```

要是知道NSArray是个类族，那就会明白上面的代码错在哪里：其中if语句永远不可能为真。[mybeAnArray class]所返回的类绝不可能是NSArray类本身，因为由NSArray的初始化方法所返回的那个实例其类型是隐藏在类族公共接口（publlic facade）后面的某个内部类型（internal type）。<br />      不过，任然有方法判断出某个实例所属的类是否位于类族中。如下：

```objectivec
 id mybeAnArray = /**/;
 if（ [mybeAnArray isKindOfClass:[NSArray class]]）{
      //will be hit
  }
```

不管创建的事可变还是不可变的数组，在**alloc**之后得到的类都是**__NSPlaceholderArray**。而当我们**init**一个不可变的空数组之后，得到的是**__NSArray0**；如果有且只有一个元素，那就是**__NSSingleObjectArrayI**；有多个元素的，叫做 **__NSArrayI**；**init**出来一个可变数组的话，都是 **__NSArrayM**。
这里**__NSSingleObjectArrayI**，需要说明它的用意：


#### __NSSingleObjectArrayI
**<br />作为对比，**__NSArrayI**必须要实现

- count<br />
- objectAtIndex:<br />

这两个个方法，但是我们可以非常显而易见的看出来，当数组只有一个数字的时候，是完全不需要这两个方法的。
再深入一点的说明一下，**NSSingleObjectArrayI是不需要去记录字符串长度的。它会比**NSArrayI少8个字节的长度。苹果可能是为了优化性能考虑，从而在iOS8之后推出这个新的子类。

#### 实例测试
不同的创建数组的方法导致不同的类簇（其实也就是不同的指针），例如：


```objectivec
NSArray *arr1 =  @[@"1", @"2"];  
NSLog(@"arr1 --- %@", [arr1 class]);

NSArray *arr2 =  [[NSArray alloc] init];
NSLog(@"arr2 --- %@", [arr2 class]);

NSArray *arr3 =  [[NSArray alloc] initWithObjects:@"1",nil];
NSLog(@"arr3 --- %@", [arr3 class]);

NSArray *arr4 =  [NSArray alloc];
NSLog(@"arr4 --- %@", [arr4 class]);

NSMutableArray *arr5 =  [NSMutableArray array];
NSLog(@"arr5 --- %@", [arr5 class]);
```


输出<br /><img src='https://static1.tuyacn.com/static/ty-lib/ios-docs/1561641369843-d628abd9-4f4c-4db1-85d1-9914ded8f631.png#align=left&display=inline&height=308&name=image-20190627211648568.png&originHeight=308&originWidth=1432&size=150951&status=done&width=1432' width=700>

### 

### unrecognized selector：
<img src='https://static1.tuyacn.com/static/ty-lib/ios-docs/1561641394348-10b8ce52-042f-4eda-bbb1-259c80ce1688.png#align=left&display=inline&height=736&name=image-20190627205113366.png&originHeight=736&originWidth=1404&size=118832&status=done&width=1404' width=900><br />
`- (NSMethodSignature *)methodSignatureForSelector:(SEL)aSelector` 处理了下面情况：

- 如果有方法签名，则正常流程；<br />
- 如果没有方法签名，则返回一个默认的方法签名，然后在`- (void)forwardInvocation:(NSInvocation *)anInvocation`中处理；<br />
- 如果子类重载了该方法，则返回nil，具体处理交给子类<br />
- 如果子类没有重载，并且是白名单的，再处理<br />


```objectivec
+ (void)load {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{

        [self tyCrash_swizzleInstanceMethodWithClass:[self class] originalSel:@selector(methodSignatureForSelector:) swizzledSel:@selector(tyCrash_methodSignatureForSelector:)];
        
        [self tyCrash_swizzleInstanceMethodWithClass:[self class] originalSel:@selector(forwardInvocation:) swizzledSel:@selector(tyCrash_forwardInvocation:)];
    });
}

- (NSMethodSignature *)tyCrash_methodSignatureForSelector:(SEL)aSelector {
    
    NSMethodSignature *ms = [self tyCrash_methodSignatureForSelector:aSelector];
    
    IMP originIMP = class_getMethodImplementation([NSObject class], @selector(methodSignatureForSelector:));
    IMP currentClassIMP = class_getMethodImplementation([self class], @selector(methodSignatureForSelector:));
    
    // 如果子类没有重载
    if (!ms && [self isWhiteList] && originIMP == currentClassIMP) {
        return [TYCrashProtection instanceMethodSignatureForSelector:@selector(TYCrashProtection)];
    } else {
        return ms;
    }
}

- (void)tyCrash_forwardInvocation:(NSInvocation *)anInvocation {
    if ([self isWhiteList]) {
        @try {
            [self tyCrash_forwardInvocation:anInvocation];
        } @catch (NSException *exception) {
            [TYCrashProtection tyCrash_logCrashWithException:exception];
        } @finally {
        }
    } else {
        [self tyCrash_forwardInvocation:anInvocation];
    }
}

- (BOOL)isWhiteList {
    BOOL isWhiteList = NO;
    NSString *className = NSStringFromClass([self class]);
    if ([className isEqualToString:@"NSNull"] || [className hasPrefix:@"TY"] || [className hasPrefix:@"TuyaSmart"]) {
        isWhiteList = YES;
    }
    
    return isWhiteList;
}
```


### 子线程刷新UI检测
在iOS中是不允许在子线程中对UI进行操作和渲染的，不然会造成未知的错误和问题，甚至会导致crash。为了对于这种情况可以提早被我们发现，在TYCrashProtection中增加了子线程UI渲染检查查询。

#### 具体事项思路
我们hook住UIView的三个必须在主线程中操作的绘制方法。

- 1、setNeedsLayout 
- 2、setNeedsDisplay 
- 3、setNeedsDisplayInRect:
- 然后判断他们是不是在子线程中进行操作,如果是在子线程进行操作的话，打印出当前代码调用堆栈，提供给开发进行解决。

具体代码如下：

```objectivec
#import "UIView+TYCrashProtection.h"
#import "NSObject+TYCrashProtection.h"

@implementation UIView (TYCrashProtection)

+ (void)load{
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        if ([NSStringFromClass([self class]) isEqualToString:@"UIView"]) {
            [self tyCrash_swizzleInstanceMethodWithClass:@selector(setNeedsLayout) swizzledSel:@selector(tyCrash_setNeedsLayout)];
            [self tyCrash_swizzleInstanceMethodWithClass:@selector(setNeedsDisplay) swizzledSel:@selector(tyCrash_setNeedsDisplay)];
            [self tyCrash_swizzleInstanceMethodWithClass:@selector(setNeedsDisplayInRect:) swizzledSel:@selector(tyCrash_setNeedsDisplayInRect:)];
        }
    });
}

- (void)tyCrash_setNeedsLayout {
    [self tyCrash_setNeedsLayout];
    [self tyCrashUICheck];
}

- (void)tyCrash_setNeedsDisplay {
    [self tyCrash_setNeedsDisplay];
    [self tyCrashUICheck];
}

- (void)tyCrash_setNeedsDisplayInRect:(CGRect)rect{
    [self tyCrash_setNeedsDisplayInRect:rect];
    [self tyCrashUICheck];
}

- (void)tyCrashUICheck {
    if(![NSThread isMainThread]) {
        NSException *exception = [NSException exceptionWithName:@"UIView Crash Protection" reason:@"sub thread update UI" userInfo:nil]
        [TYCrashProtection tyCrash_logCrashWithException:exception];
    }
}
```
