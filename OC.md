### OC常见题

#### 1、分类(category)和类扩展(extension)的区别？

```c++
//Category
//Category 是表示一个指向分类的结构体的指针，其定义如下：
typedef struct objc_category *Category;
struct objc_category {
  char *category_name                          OBJC2_UNAVAILABLE; // 分类名
  char *class_name                             OBJC2_UNAVAILABLE; // 分类所属的类名
  struct objc_method_list *instance_methods    OBJC2_UNAVAILABLE; // 实例方法列表
  struct objc_method_list *class_methods       OBJC2_UNAVAILABLE; // 类方法列表
  struct objc_protocol_list *protocols         OBJC2_UNAVAILABLE; // 分类所实现的协议列表
}
```

通过上面我们可以发现，这个结构体主要包含了分类定义的实例方法与类方法，其中instance_methods 列表是 objc_class 中方法列表的一个子集，而class_methods列表是元类方法列表的一个子集。 但这个结构体里面，根本没有属性列表。

分类是用于给原有类添加方法的，因为分类的结构体指针中，没有属性列表，只有方法列表。所以**< 原则上讲它只能添加方法， 不能添加属性(成员变量)，实际上可以通过其它方式添加属性>** 

我们知道在一个类中用@property声明属性，编译器会自动帮我们生成成员变量和setter/getter，但分类的指针结构体中，根本没有属性列表。所以在分类中用@property声明属性，既无法生成成员变量也无法生成setter/getter。

我们可以通过runtime的`objc_setAssociatedObject` 和 `objc_getAssociatedObject` 实现setter和getter方法，给分类添加属性，但调用成员变量依然报错。

Extension是Category的一个特例。类扩展与分类相比只少了分类的名称，所以称之为“匿名分类”。

```objectivec
@interface XXX ()
//私有属性
//私有方法（如果不实现，编译时会报警,Method definition for 'XXX' not found）
@end
```

### 分类与类扩展的区别：

①类别中原则上只能增加方法（能添加属性的的原因只是通过`runtime`解决无`setter/getter`的问题而已）；

②类扩展不仅可以增加方法，还可以增加实例变量（或者属性），只是该实例变量默认是@private类型的（使用范围只能在自身类，而不是子类或其他地方）；

③类扩展中声明的方法没被实现，编译器会报警，但是类别中的方法没被实现编译器是不会有任何警告的。这是因为**类扩展是在编译阶段被添加到类中，而类别是在运行时添加到类中**。

④类扩展不能像类别那样拥有独立的实现部分（@implementation部分），也就是说，类扩展所声明的方法必须依托对应类的实现部分来实现。

⑤定义在 .m 文件中的类扩展方法为私有的，定义在 .h 文件（头文件）中的类扩展方法为公有的。类扩展是在 .m 文件中声明私有方法的非常好的方式。



参考链接：https://www.jianshu.com/p/9e827a1708c6



#### 2、对象，类对象，元类，跟元类结构体的组成以及他们是如何相关联的？

- 对象的结构体当中存放着isa指针和成员变量，isa指针指向类对象

- 类对象的isa指针指向元类，元类的isa指针指向NSObject的元类

- 类对象和元类的结构体有isa，superClass，cache等等

  

#### 3、什么是野指针？

野指针就是指向一个被释放或者被收回的对象，但是对指向该对象的指针没有做任何修改，以至于该指针让指向已经回收后的内存地址。



#### 4、不使用第三方，如何知道已经上线的App崩溃问题， 具体到哪一个类的哪一个方法的？

- Xcode -> Window -> Organizer -> Crashes

- 使用`NSSetUncaughtExceptionHandle`可以获取崩溃信息，将崩溃信息拼接好保存到沙盒中，下次启动App检查本地是否有崩溃文件，有的话上传服务器，上传成功后删除文件。

  ```objective-c
  - (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {
      // Override point for customization after application launch.
      NSSetUncaughtExceptionHandler(&my_uncaught_exception_handler);
      return YES;
  }
  
  static void my_uncaught_exception_handler (NSException *exception) {
      //这里可以取到 NSException 信息
      NSLog(@"***********************************************");
      NSLog(@"%@"，exception);
      NSLog(@"%@"，exception.callStackReturnAddresses);
      NSLog(@"%@"，exception.callStackSymbols);
      NSLog(@"***********************************************");
  }
  ```

- 可以通过runtime的置换方法，把数组越界，dictionary添加nil对象等问题方法置换，手动抛出Exception，然后上报服务器，也避免了程序崩溃。



#### 5、==、 isEqualToString、isEqual区别？

- ==：对于基本数据类型，比较的是值; 对于对象类型，比较的是对象的地址是否相同
- isEqualToString：比较两个字符串是否相等
- isEqual：判断两个对象的等同性，首先判断两个对象的地址是否相同，再判断类型是否一致， 然后再判断对象的具体内容是否一致



#### 6、*class方法和object_getClass方法有什么区别?（有待商榷）*

- 实例class方法直接返回object_getClass(self)
- 类class直接返回self
- 而object_getClass(类对象)，则返回的是元类



#### 7、NSString类型为什么要用copy修饰 ？

主要是防止NSString被修改，如果没有修改的时候用Strong也行。当String对象MyStr使用Strong修饰时，如果此时赋值给一个NSMutableString对象StrM，当StrM发生修改时，MyStr也会跟着修改，容易引出bug。



#### 8、深拷贝和浅拷贝

- 浅拷贝就是拷贝后，并没有进行真正的复制，而是复制的对象和原对象都指向同一个地址
- 深拷贝是真正的复制了一份，复制的对象指向了新的地址

- copy： 对于可变对象为深拷贝，对于不可变对象为浅拷贝

- mutableCopy：始终是深拷贝



#### 9、全局变量、静态变量、局部变量、自动变量的区别？

- 全局变量

  - ```objective-c
    #import "ViewController.h"
    int count = 10;
    @interface ViewController ()
    @end
    ```

  - 函数外面声明
  - 可以跨文件访问
  - 可以在声明时赋上初始值
  - 存储位置：既非堆，也非栈，而是专门的【全局（静态）存储区static】！

- 静态变量

  - ```objective-c
    #import "ViewController.h"
    static int count = 10;
    @interface ViewController ()
    @end
    ```

  - 函数外面 或 内部声明（即可修饰原全局变量亦可修饰原局部变量）
  - 仅声明该变量的文件可以访问
  - 可以在声明时赋上初始值
  - 存储位置：既非堆，也非栈，而是专门的【全局（静态）存储区static】！

- 局部变量（自动变量）

  - ```objective-c
    - (int)getNumber {
        int a = 10;
        return a;
    }
    ```

  - 函数内部声明
  - 仅当函数执行时存在
  - 仅在本文件本函数内可访问
  - 存储位置：自动保存在函数的每次执行的【栈帧】中，并随着函数结束后自动释放，另外，函数每次执行则保存在【栈】中



#### 10、iOS中block 捕获外部局部变量实际上发生了什么？__block 中又做了什么？

- 对于普通的auto局部变量（栈变量），Block捕获时，将值拷贝进Block用结构体的成员变量中。因此后续对局部变量的改变就再也影响不了Block外部。
- 对于__block修饰的局部变量，Block捕获时，记录了该变量的地址。所以后续该变量的值改变了，block调用时，通过地址获取到的值仍然是最新的值。



#### 11、block分为哪几种？区别是什么？

OC中，一般Block就分为以下3种，`_NSConcreteStackBlock，_NSConcreteMallocBlock，_NSConcreteGlobalBlock`。

- _NSConcreteStackBlock：
  只用到外部局部变量（基本数据、OC对象）、成员属性变量，且没有强指针引用的block。当函数返回时会被销毁

- _NSConcreteMallocBlock：
  有强指针引用或copy修饰的成员属性引用的block会被复制一份到堆中成为MallocBlock，没有强指针引用即销毁，生命周期由程序员控制

- _NSConcreteGlobalBlock：
  没有访问外部局部变量（基本数据、OC对象）、成员属性变量或只用到全局变量、静态变量（局部或者全局静态变量）



#### 12、block为什么用copy修饰？

- block 是一个对象
- block使用了外部局部变量，这种情况也正是我们平时所常用的方式。MRC：Block的内存地址显示在栈区，栈区的特点就是创建的对象随时可能被销毁，一旦被销毁后续再次调用空对象就可能会造成程序崩溃，在对block进行copy后，block存放在堆区。所以在使用Block属性时使用copy修饰，但是ARC中的Block都会在堆上的，系统会默认对Block进行copy操作
- 使用strong也可以，但是block的strong行为默认是用copy的行为实现的，因为block变量默认是声明为栈变量的，为了能够在block的声明域外使用，所以要把block拷贝（copy）到堆，所以说为了block属性声明和实际的操作一致，最好声明为copy。



#### 13、load 和 initilze 的调用情况，以及子类的调用顺序问题？

- initialize 方法是这个类第一次调用（比如init）时会调用此方法，并且只会调用一次。 如果某一个类一直没有被用到，此方法也不会执行。

- initialize先初始化父类， 在初始化子类，子类的initialize 会覆盖父类的方法。

- 分类中实现了initialize会覆盖本来的initialize方法，如果多个分类都执行了initialize ，那么只是执行最后编译的那个。

  

- load当程序被加载的时候就会调用， 其加载顺序为， 如果子类实现类load 先执行父类 -> 在执行子类，而分类的在最后执行。
- 如果子类不实现load，父类的load就不会被执行。
- load是线程安全的，其内部使用了锁，所以我们应该避免在load方法中线程阻塞。
- load在分类中，重写了load方法， 不会影响其主类的方法。即不会覆盖本类的load方法
- 当有多个类的时候，每个类的load的执行顺序和编译顺序一致。



#### 14、RunLoop是什么？

- RunLoop 就是一个事件处理的循环，用来不停的调度工作以及处理输入事件。使用 RunLoop 的目的是让你的线程在有工作的时候忙于工作，而没工作的时候处于休眠状态。 runloop 的设计是为了减少 cpu 无谓的空转。

- RunLoop的作用就是用来管理线程的， 当线程的RunLoop开启之后，线程就会在执行完成任务后，进入休眠状态，随时等待接收新的任务，而不是退出。

- 内部其实就是一个do-while循环，在这个循环内部不断的处理各种任务（比如Source、timer、Observer）。

- 一个线程对应一个runloop，主线程的runloop默认开启，子线程的runloop需要手动调用run()方法。

- runloop只能选择一个mode，如果当前mode没有任何事件（source0、source1、timer）则直接退出runloop。

  ```objective-c
  //1.获得子线程对应的runloop
      NSRunLoop *runloop = [NSRunLoop currentRunLoop];
      
      //保证runloop不退出
      //NSTimer *timer = [NSTimer timerWithTimeInterval:2.0 target:self selector:@selector(run) userInfo:nil repeats:YES];
      //[runloop addTimer:timer forMode:NSDefaultRunLoopMode];
      [runloop addPort:[NSPort port] forMode:NSDefaultRunLoopMode];
      
      //2.默认是没有开启
      [runloop runUntilDate:[NSDate dateWithTimeIntervalSinceNow:10]];
  ```
  
  
  
  

#### 15、Runloop模式有哪些？

1. **kCFRunLoopDefaultMode**：App的默认Mode，通常主线程是在这个Mode下运行
2. **UITrackingRunLoopMode**：界面跟踪 Mode，用于 ScrollView 追踪触摸滑动，保证界面滑动时不受其他 Mode 影响
3. **UIInitializationRunLoopMode**: 在刚启动 App 时第进入的第一个 Mode，启动完成后就不再使用，会切换到kCFRunLoopDefaultMode
4. **GSEventReceiveRunLoopMode**: 接受系统事件的内部 Mode，通常用不到
5. **kCFRunLoopCommonModes**: 这是一个占位用的Mode，作为标记kCFRunLoopDefaultMode和UITrackingRunLoopMode用，并不是一种真正的Mode



#### 16、用weak修饰的对象释放后为什么会自动为nil？

runtime会把weak对象放入一个hash表中，Key是weak所指对象的地址，Value是weak指针的地址（这个地址的值是所指对象指针的地址）数组。

weak 的实现原理可以概括以下三步：

1、初始化时：runtime会调用objc_initWeak函数，初始化一个新的weak指针指向对象的地址。

2、添加引用时：objc_initWeak函数会调用 objc_storeWeak() 函数， objc_storeWeak() 的作用是更新指针指向，创建对应的弱引用表。

3、释放时，调用clearDeallocating函数。clearDeallocating函数首先根据对象地址获取所有weak指针地址的数组，然后遍历这个数组把其中的数据设为nil，最后把这个entry从weak表中删除，最后清理对象的记录。



#### 17、@synthesize 和 @dynamic 分别有什么作用

@property有两个对应的词，一个是 @synthesize，一个是 @dynamic。如果 @synthesize和 @dynamic都没写，那么默认的就是@syntheszie var = _var;

- `@synthesize` 的语义是如果你没有手动实现 setter 方法和 getter 方法，那么编译器会自动为你加上这两个方法
- `@dynamic` 告诉编译器：属性的 setter 与 getter 方法由用户自己实现，不自动生成。



#### 18、UIView和CALayer的区别和联系

- 两者最明显的区别是 View可以接受并处理事件，而 Layer 不可以

- 每个 UIView 内部都有一个 CALayer 在背后提供内容的绘制和显示，并且 UIView 的尺寸样式都由内部的 Layer 所提供。两者都有树状层级结构，layer 内部有 SubLayers，View 内部有 SubViews.但是 Layer 比 View 多了个AnchorPoint 
- 在 View显示的时候，UIView 做为 Layer 的 CALayerDelegate，View 的显示内容由内部的 CALayer 的 display 
- CALayer 是默认修改属性支持隐式动画的，在给 UIView 的 Layer 做动画的时候，View 作为 Layer 的代理，Layer 通过 actionForLayer:forKey:向 View请求相应的 action(动画行为) 
- layer 内部维护着三分 layer tree，分别是 presentLayer Tree(动画树)，modeLayer Tree(模型树)， Render Tree (渲染树)，在做 iOS动画的时候，我们修改动画的属性，在动画的其实是 Layer 的 presentLayer的属性值，而最终展示在界面上的其实是提供 View的modelLayer



#### 19、 static有什么作用?

static关键字可以修饰函数和变量，作用如下：

-  **隐藏** 通过static修饰的函数或者变量，其他文件中的方法和函数无法访问
- **静态变量** 类方法不可以访问实例变量（函数），通过static修饰的实例变量（函数），可以被类 方法访问； 
- **持久** static修饰的变量，能且只能被初始化一次； 
- **默认初始化** static修饰的变量，默认初始化为0；



#### 20、KVO底层实现原理

当对象某个属性添加了KVO属性观察：

```objective-c
Person *p1 = [[Person alloc] init];
NSLog(@"KVO 之前：%@", object_getClass(p1)); //KVO 之前：Person
[p1 addObserver:self forKeyPath:@"name" options:NSKeyValueObservingOptionNew context:nil];
NSLog(@"KVO 之后：%@", object_getClass(p1)); //KVO 之后：NSKVONotifying_Person
```

此时系统会动态生成一个类名为NSKVONotifying_Person的Person 的子类，然后把Person类的isa指针指向NSKVONotifying_Person子类，并且系统修改了默认方法实现，把set方法(setName:)的IMP指针指向了 Foundation 框架里的` _NSSetObjectValueAndNotify` 函数。

Person内部只有setName: 方法。

而NSKVONotifying_Person内部有：1、setName: 方法；2、还重写了class和dealloc方法；3、`_isKVOA`方法

大致的得出， p1添加 KVO 后 runtime 动态的生成了一个 NSKVONotifying_Person子类 并重写了 setName 方法

setName 方法内部调用了 Foundation 的` _NSSetObjectValueAndNotify` 函数 ,在函数 

`_NSSetObjectValueAndNotify` 内部

1、首先会调用 willChangeValueForKey
2、然后给 name 属性赋值
3、最后调用 didChangeValueForKey
4、最后调用 observer 的 observeValueForKeyPath 去告诉监听器属性值发生了改变 



大致实现如下：

```objective-c
@implementation NSKVONotifying_Person

- (void)setName:(NSString *)name {
    _NSSetObjectValueAndNotify();
}

- (void)willChangeValueForKey:(NSString *)key {
    NSLog(@"willChangeValueForKey");
    [super willChangeValueForKey:key];
}

- (void)didChangeValueForKey:(NSString *)key {
    NSLog(@"didChangeValueForKey");
    [super didChangeValueForKey:key];
}

void _NSSetObjectValueAndNotify() {
    
    [self willChangeValueForKey: @"name"];
    [super setName: name];
    [self didChangeValueForKey: @"name"];
}

- (Class)class {
    return class_getSuperclass(object_getClass(self));
}
@end
```

至于重写了 dealloc 和 class 方法 是为了做一些 KVO 释放内存 和 隐藏外界对于 NSKVONotifying_Person 子类的存在。



#### 21、iOS的两种渲染

**当前屏幕渲染(On-Screen Rendering)：**

- 渲染发生在当前用于显示的屏幕缓冲区中。
- App要将渲染的信息提交给CPU，CPU经过计算完成之后提交给GPU，CPU将内容渲染完成后放到帧缓冲区（FrameBuffer）中，最后显示到屏幕上。



**离屏渲染（Off-Screem Rendering）：**

- 渲染发生在当前屏幕之外。
- 与正常渲染流程不同的是，CPU把渲染好的内容临时保存到离屏渲染缓冲区（offscreen-buffer）中，等到要用时再取出（一般需要切换上下文）再提交到帧缓冲区（FrameBuffer）中。

**性能问题：**

- 相比于正常的渲染流程，离屏渲染需要额外创建一个缓冲区，需要多耗费一些空间。
- 触发离屏渲染后，需要先从 Frame Buffer 切换到 Off-Screen Buffer ，渲染完毕后再切换回 Frame Buffer ，这一过程需是比较耗费性能的，因为要来回切换上下文。

**触发方式：**

- 对layer层相关处理：包括圆角、阴影、mask等。iOS系统扁平化后出现的高斯模糊也是利用离屏渲染方式。
- shouldRasterize（光栅化）是一种主动行为，是为了提高复用的效率。通常是设置layer的shouldRasterize = true来实现。

**离屏渲染逻辑**：

图层的叠加绘制大致遵循“画家算法”，也就是由远到近的方式将图层绘制到屏幕上，绘制近距离图层会有覆盖远图层的逻辑。

- 普通渲染方式，绘制完一层图层后，直接舍弃掉。紧接着绘制稍近的图层。
- 而进行离屏渲染方式（圆角、剪裁等操作）时，每层图层绘制完不会马上删除，而是先保存在离屏缓存区中，等图层绘制完成后，再依次进行特殊处理（圆角、剪裁等操作）

**为什么还要用？**

- 存在一些特殊效果，正常流程无法完成，必须使用离屏渲染，比如圆角、阴影和遮罩、高斯模糊、半透明图层混合等
- 有些后续经常用到的图层数据，可以先缓存在离屏缓存，用到时直接复用



#### 22、iOS组件化方案的三种实现

组件化的三种方案：`url-block`、`protocol-class`、`target-action`

- **url-block**：这是蘑菇街中应用的一种页面间调用的方式，通过在启动时注册组件提供的服务，把调用组件使用的`url`和组件提供的服务`block`对应起来，保存到内存中。在使用组件的服务时，通过`url`找到对应的`block`，然后获取服务。

  - **注册：**

    ```objective-c
    [MGJRouter registerURLPattern:@"mgj://detail?id=:id" toHandler:^(NSDictionary *routerParameters) {
        NSNumber *id = routerParameters[@"id"];
        // create view controller with id
        // push view controller
    }];
    ```

  - **调用：**

    ```objective-c
    [MGJRouter openURL:@"mgj://detail?id=404"]
    ```

  - **存在问题：**

    - 需要在内存中维护`url-block`的表，组件多了可能会有内存问题

    - `url`的参数传递受到限制，只能传递常规的字符串参数，无法传递非常规参数，如`UIImage`、`NSData`等类型

    - 没有区分本地调用和远程调用的情况，尤其是远程调用，会因为`url`参数受限，导致一些功能受限

    - 组件本身依赖了中间件，且分散注册使的耦合较多

    

- **protocol-class：**通过`protocol`定义服务接口，组件通过实现该接口来提供接口定义的服务，具体实现就是把`protocol`和`class`做一个映射，同时在内存中保存一张映射表，使用的时候，就通过`protocol`找到对应的`class`来获取需要的服务。

  - **注册：**

    ```objective-c
    [ModuleManager registerClass:ClassA forProtocol:ProtocolA]
    ```

  - **调用：**

    ```objective-c
    [ModuleManager classForProtocol:ProtocolA]
    ```

  - **存在问题：**

    解决了`url-block`中无法传递非常规参数的问题，使得组件间的调用更为方便，但是它依然没有解决组件依赖中间件的问题、内存中维护映射表的问题、组件的分散调用的问题。设计思想和方案一类似，都是通过给组件加了一层`wrapper`，然后给使用者调用。



- **target-action：**通过给组件包装一层`wrapper`来给外界提供服务，然后调用者通过依赖中间件来使用服务；其中，中间件是通过`runtime`来调用组件的服务，是真正意义上的解耦，也是该方案最核心的地方。具体实施过程是给组件封装一层`target`对象来对外提供服务，不会对原来组件造成入侵；然后，通过实现中间件的`category`来提供服务给调用者，这样使用者只需要依赖中间件，而组件则不需要依赖中间件。

  - **target:**

    ```objective-c
    @interface Target_ModulAViewController : UIViewController
    - (UIViewController *)Action_ModulAViewController:(NSDictionary *)param;
    @end
    ```

  - **CTMediator分类：**

    ```objective-c
    //CTMediator+ModulA.m
    @implementation CTMediator (ModulA)
    // 公开此方法
    - (UIViewController *)modulAWithShowPicture:(BOOL)showPicture
    {
        NSDictionary *dic = @{@"showPicture":@(showPicture)};
        return [self performTarget:@"ModulAViewController" action:@"ModulAViewController" params:dic shouldCacheTarget:NO];
    }
    @end
    ```

  - **调用：**

    ```objective-c
    //ViewController.m
    #import <ModulCategotyA/CTMediator+ModulA.h>
    - (void)doneAction {
        UIViewController *vc = [[CTMediator sharedInstance] modulAWithShowPicture:YES];
        [self.navigationController pushViewController:vc animated:YES];
    }
    ```

    

#### 22、iOS自动释放池

自动释放池是oc提供的一种自动回收的机制，具有延迟释放的特性，即当我们创建了一个对象，并把他加入到了自动释放池中时，他不会立即被释放，会等到一次runloop结束或者作用域超出{}或者超出[pool release]之后再被释放。

系统通过一个栈来管理所有的自动释放池，每当创建了一个新的自动释放池，系统就会把它压入栈顶，每当有一个自动释放池要被释放的时候，就会从栈顶弹出一个自动释放池，该自动释放池会对加入他中的对象做一次release，然后销毁，栈顶下移，每当有一个新对象被创建，他会被加入到离他最近的自动释放池，也就是位于当前栈顶的自动释放池。

**MRC：**

- 通过手动创建的方式来创建自动释放池

  ```objective-c
  NSAutoreleasePool *pool = [[ NSAutoreleasePool alloc]init ];//创建一个自动释放池
  Person *person = [[Person alloc]init];
  //调autorelease方法将对象加入到自动释放池
  //注意使用该方法，对象不会自己加入到自动释放池，需要人为调用autorelease方法加入
  [person autorelease];
  //,手动释放自动释放池执行完这行代码是，自动释放池会对加入他中的对象做一次release操作
  [pool release];
  ```

  自动释放池销毁时机：[pool release]代码执行完后

- 通过@autoreleasepool来创建

  ```objective-c
  @autoreleasepool {
          //在这个{}之内的变量默认被添加到自动释放池
           Person *p = [[Person alloc] init];
        }//出了这个括号，p被释放
  ```

  系统就是通过@autoreleasepool {}这种方式来为我们创建自动释放池的，一个线程对应一个runloop，系统会为每一个runloop隐式的创建一个自动释放池，所有的autoreleasePool构成一个栈式结构，在每个runloop结束时，当前栈顶的autoreleasePool会被销毁，而且会对其中的每一个对象做一次release（严格来说，是你对这个对象做了几次autorelease就会做几次release，不一定是一次)

**ARC：**

- ARC下除了NSAutoreleasePool不可用以外，其他的同MRC

**自动释放池的原理：**每一个自动释放池没有单独的结构，每一个autorealeasePool对象都是由若干个个autoreleasePoolPage通过双向链表连接而成，当一个对象调用了autorelease方法，这个对象就会被加入到当前自动释放池的最新的autoreleasePoolPage中。

```objective-c
class AutoreleasePoolPage {
    magic_t const magic;
    id *next;//指向栈顶最新被添加进来的autorelease对象的下一个位置
    pthread_t const thread;//指向当前线程的
    AutoreleasePoolPage * const parent;
    AutoreleasePoolPage *child;
    uint32_t const depth;
    uint32_t 
  }
```

- autoreleasePoolPage是按照线程一一对应的，
- autoreleasePoolPage会开辟4096字节空间，除了上面的实例变量所占的空间，剩余的空间全部用来存储autorelease对象的地址
- id *next:指向栈顶最新被添加进来的autorelease对象的下一个位置
- 一个autoreleasePoolPage空间被占满时，会创建一个新的autoreleasePoolPage对象，后来的对象添加在在新的autoreleasePoolPage中
- 每当进行一objc_autoreleasePoolPush（）次入栈调用时，runtime会像当前的autoreleasePoolPage中添加一个哨兵对象，值为0，objc_autoreleasePoolPush（）会返回这个哨兵对象的地址，然后将该参数作为objc_autoreleasePoolPop（）的参数，根据哨兵对象的地址找到他所在的page，哨兵对象可以跨越到不通的page，直到找到他所在的page，在当前page中将所有晚于哨兵对象的autorelease对象都做一次release，并移动next指针到正确位置。



#### 23、iOS中的锁

- 自旋锁（`OSSPinLock`、`os_unfair_lock`）
- 互斥锁（`pthread_mutex_t`、`NSLock`、`@synthronized`）
- 递归锁（`pthread_mutex_t`、`NSRecursiveLock`）
- 条件锁（`pthread_cond_t`、`NSCondition`、`NSConditionLock`）
- 信号量（`dispatch_semaphore_t`、`pthread_mutex_t`等）
- 读写锁（`pthread_rwlock_t`）
- 栅栏（`dispatch_barrier_sync`、`dispatch_barrier_async`）
- atomic

**自旋锁和互斥锁有什么区别？**

**自旋锁：**就是当资源被占用时，其他线程访问时会使其不停地循环访问，导致新访问者一直处于忙碌状态，直到资源持有者释放锁。也正是因为其他调用者会保持自旋状态，使得在锁的保持者释放锁时能够即刻获得锁，效率非常高。同时相对而言也会比较消耗CPU资源。

**互斥锁：**互斥锁和自旋锁类似，当资源被访问时，其他线程是无法同时访问的。不同的是互斥锁在被占用的情况下，只能进入休眠状态，等锁被释放后，CPU会唤醒资源调用者。



#### 24、iOS中显式和隐式动画的区别

- 隐式动画一直存在，是由Core Animation自动完成的，ore Animation在每个runloop周期中自动开始一次新的事务，即使你不显式的用[CATransaction begin]开始一次事务，任何在一次runloop循环中属性的改变都会被集中起来，然后做一次0.25秒的动画。
- 显式动画，Core Animation提供的显式动画类型，既可以直接对图层属性做动画，也可以覆盖默认的图层行为。我们经常使用的CABasicAnimation，CAKeyframeAnimation，CATransitionAnimation，CAAnimationGroup等都是显式动画类型，这些CAAnimation类型可以直接提交到CALayer上。

























腾讯面试：什么是链表？ 链表和数组的区别？什么是runloop？怎么保证线程安全？手写合并两个升序链表？对1w首歌曲进行排序？MVVM的viewmodel做了什么？Rxswift的弊端（真不知道）？



















