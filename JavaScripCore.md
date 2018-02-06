##相关类

JSContext：JS执行环境，通过JSVirtualMachine管理对象生命周期
JSValue：对JS的操作、OC和JS对象的转换 都通过它，一个JSValue对象强引用一个JSContext对象
JSManagedValue：JS和OC对象的内存管理辅助对象。JS是垃圾回收机制，且JS中的对象都是强引用；而OC是引用计数机制，如果双方相互引用，势必会造成循环引用，导致内存泄露。可以用JSManagedValue保存JSValue来避免。
JSVirtualMachine：JS虚拟机（运行环境），有独立的堆空间和垃圾回收机制
JSExport：想在JS中使用OC对象（对象、方法、属性），OC对象需事先JSExport协议（或使用block方式#brilliant）

##OC与JS通信

###OC call JS：
```
    self.context = [[JSContext alloc] init];

    NSString *js = @"function add(a,b) {return a+b}";
    
    [self.context evaluateScript:js];
    
    JSValue *n = [self.context[@"add"] callWithArguments:@[@2, @3]];

    NSLog(@"---%@", @([n toInt32]));//---5
```

###JS call OC：

####block方式：
```
    self.context = [[JSContext alloc] init];
    
    self.context[@"add"] = ^(NSInteger a, NSInteger b) {
        NSLog(@"---%@", @(a + b));
    };
    
    [self.context evaluateScript:@"add(2,3)"];
```

####JSExport协议方式：
```
//定义一个JSExport protocol
@protocol JSExportTest <JSExport>

- (NSInteger)add:(NSInteger)a b:(NSInteger)b;

@property (nonatomic, assign) NSInteger sum;

@end

//建一个对象去实现这个协议：

@interface JSProtocolObj : NSObject<JSExportTest>
@end

@implementation JSProtocolObj
@synthesize sum = _sum;
//实现协议方法
- (NSInteger)add:(NSInteger)a b:(NSInteger)b
{
    return a+b;
}
//重写setter方法方便打印信息，
- (void)setSum:(NSInteger)sum
{
    NSLog(@"--%@", @(sum));
    _sum = sum;
}

@end

//在VC中进行测试
@interface ViewController () <JSExportTest>

@property (nonatomic, strong) JSProtocolObj *obj;
@property (nonatomic, strong) JSContext *context;

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    //创建context
    self.context = [[JSContext alloc] init];
    //设置异常处理
    self.context.exceptionHandler = ^(JSContext *context, JSValue *exception) {
        [JSContext currentContext].exception = exception;
        NSLog(@"exception:%@",exception);
    };
    //将obj添加到context中
    self.context[@"OCObj"] = self.obj;
    //JS里面调用Obj方法，并将结果赋值给Obj的sum属性
    [self.context evaluateScript:@"OCObj.sum = OCObj.addB(2,3)"];
    
}
```

要注意的是OC的函数命名和JS函数命名规则问题。协议中定义的是add: b:，但是JS里面方法名字是addB(a,b)。可以通过JSExportAs这个宏转换成JS的函数名字。

```
@protocol JSExportTest <JSExport>
//用宏转换下，将JS函数名字指定为add；
JSExportAs(add, - (NSInteger)add:(NSInteger)a b:(NSInteger)b);

@property (nonatomic, assign) NSInteger sum;

@end
```

##内存管理：

OC使用ARC（引用计数）
JS使用GC（垃圾回收），且所有对象都是强引用，但GC会帮它打破循环引用
使用 JavaScriptCore 里提供的 API ，正常情况下无需关注OC 和 JS 对象的内存管理问题。但是还是有几个问题需要注意：

###1、不要在block里面直接使用context，或者使用外部的JSValue对象。
```
// 错误示例
// e.g.1
    self.context[@"block"] = ^(){
     JSValue *value = [JSValue valueWithObject:@"aaa" inContext:self.context];
    };

// e.g.2
    JSValue *value = [JSValue valueWithObject:@"ssss" inContext:self.context];
    
    self.context[@"log"] = ^(){
        NSLog(@"%@",value);
    };
```

上例2：value 强引用 context，context 又强引用了value

正确（通常）做法如下：
```
//正确的做法，str对象是JS那边传递过来。
self.context[@"log"] = ^(NSString *str){
        NSLog(@"%@",str);
    };
```


###2、OC对象不要用属性直接保存JSValue对象，容易循环引用

```
// 错误示例

//定义一个JSExport protocol
@protocol JSExportTest <JSExport>
//用来保存JS的对象
@property (nonatomic, strong) JSvalue *jsValue;

@end

//建一个对象去实现这个协议：

@interface JSProtocolObj : NSObject<JSExportTest>
@end

@implementation JSProtocolObj

@synthesize jsValue = _jsValue;

@end

//在VC中进行测试
@interface ViewController () <JSExportTest>

@property (nonatomic, strong) JSProtocolObj *obj;
@property (nonatomic, strong) JSContext *context;

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    //创建context
    self.context = [[JSContext alloc] init];
    //设置异常处理
    self.context.exceptionHandler = ^(JSContext *context, JSValue *exception) {
        [JSContext currentContext].exception = exception;
        NSLog(@"exception:%@",exception);
    };
   //加载JS代码到context中
   [self.context evaluateScript:
   @"function callback (){};
   
    function setObj(obj) {
    this.obj = obj;
    obj.jsValue=callback;
}"];
   //调用JS方法
   [self.context[@"setObj"] callWithArguments:@[self.obj]];  
}
```

JS 保存了 OC的 obj 对象，obj保存了JS 的 callback 对象，引起循环引用；
解决思路：打破引用。使用内存管理辅助对象JSManagedValue

JSManagedValue 本身就是我们需要的弱引用。用官方的话来说叫garbage collection weak reference。但是它帮助我们持有JSValue对象必须同时满足一下两个条件：

* The JSManagedValue's JavaScript value is reachable from JavaScript
JSManagedValue 中的 JS 对象必须可以在 JavaScript 中访问（like，事先协议或用block声明）

* The owner of the managed reference is reachable in Objective-C. Manually adding or removing the managed reference in the JSVirtualMachine determines reachability.
JSManagedValue 的 owner 必须能在OC 中访问。在JSVirtualMachine 中 手动 添加/删除 的JSManagedValue是OC可达的（可被OC访问）

意思是：JSManagedValue 帮助我们保存JSValue，那里面保存的JS对象必须在JS中存在，同时 JSManagedValue 的owner在OC中也存在。

可以通过它提供的两个方法创建JSManagedValue对象:
```
+ (JSManagedValue *)managedValueWithValue:(JSValue *)value;
+ (JSManagedValue *)managedValueWithValue:(JSValue *)value andOwner:(id)owner
```

通过JSVirtualMachine的方法 手动建立/移除 弱引用关系:
```
// 建立这个弱引用关系
- (void)addManagedReference:(id)object withOwner:(id)owner
// 手动移除他们之间的联系
- (void)removeManagedReference:(id)object withOwner:(id)owner
```

修改后代码如下：
```
//定义一个JSExport protocol
@protocol JSExportTest <JSExport>
//用来保存JS的对象
@property (nonatomic, strong) JSValue *jsValue;

@end

//建一个对象去实现这个协议：

@interface JSProtocolObj : NSObject<JSExportTest>
//添加一个JSManagedValue用来保存JSValue
@property (nonatomic, strong) JSManagedValue *managedValue;

@end

@implementation JSProtocolObj

@synthesize jsValue = _jsValue;
//重写setter方法
- (void)setJsValue:(JSValue *)jsValue
{
    _managedValue = [JSManagedValue managedValueWithValue:jsValue];
    
    [[[JSContext currentContext] virtualMachine] addManagedReference:_managedValue 
    withOwner:self];
}

@end

//在VC中进行测试
@interface ViewController () <JSExportTest>

@property (nonatomic, strong) JSProtocolObj *obj;
@property (nonatomic, strong) JSContext *context;

@end

@implementation ViewController

- (void)viewDidLoad {
    [super viewDidLoad];
    //创建context
    self.context = [[JSContext alloc] init];
    //设置异常处理
    self.context.exceptionHandler = ^(JSContext *context, JSValue *exception) {
        [JSContext currentContext].exception = exception;
        NSLog(@"exception:%@",exception);
    };
   //加载JS代码到context中
   [self.context evaluateScript:
   @"function callback (){}; 
   
   function setObj(obj) {
   this.obj = obj;
   obj.jsValue=callback;
   }"];
   //调用JS方法
   [self.context[@"setObj"] callWithArguments:@[self.obj]];  
}
```

###3、不要在不同的 JSVirtualMachine 之间进行传递JS对象

一个 JSVirtualMachine可以运行多个context，由于都是在同一个 堆内存 和同一个 垃圾回收 下，所以相互之间传值是没问题的。但是如果在不同的 JSVirtualMachine传值，垃圾回收就不知道他们之间的关系了，可能会引起异常。

##线程

