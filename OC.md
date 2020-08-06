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

分类是用于给原有类添加方法的,因为分类的结构体指针中，没有属性列表，只有方法列表。所以**< 原则上讲它只能添加方法, 不能添加属性(成员变量),实际上可以通过其它方式添加属性>** 

我们知道在一个类中用@property声明属性，编译器会自动帮我们生成成员变量和setter/getter，但分类的指针结构体中，根本没有属性列表。所以在分类中用@property声明属性，既无法生成成员变量也无法生成setter/getter。

我们可以通过runtime的`objc_setAssociatedObject` 和 `objc_getAssociatedObject` 实现setter和getter方法，给分类添加属性，但调用成员变量依然报错。

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
      NSLog(@"%@",exception);
      NSLog(@"%@",exception.callStackReturnAddresses);
      NSLog(@"%@",exception.callStackSymbols);
      NSLog(@"***********************************************");
  }
  ```



#### 5、==、 isEqualToString、isEqual区别？

- ==：对于基本数据类型，比较的是值; 对于对象类型，比较的是对象的地址是否相同
- isEqualToString：比较两个字符串是否相等
- isEqual：判断两个对象的等同性，首先判断两个对象的地址是否相同，再判断类型是否一致， 然后再判断对象的具体内容是否一致



#### 6、*class方法和object_getClass方法有什么区别?（有待商榷）*

- 实例class方法直接返回object_getClass(self)
- 类class直接返回self
- 而object_getClass(类对象)，则返回的是元类



#### 7、NSString类型为什么要用copy修饰 ？

主要是防止NSString被修改，如果没有修改的说法用Strong也行。当String对象MyStr使用Strong修饰时，如果此时赋值给一个NSMutableString对象StrM，当StrM发生修改时，MyStr也会跟着修改，容易引出bug。



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
- block使用了外部局部变量,这种情况也正是我们平时所常用的方式。MRC：Block的内存地址显示在栈区，栈区的特点就是创建的对象随时可能被销毁，一旦被销毁后续再次调用空对象就可能会造成程序崩溃，在对block进行copy后，block存放在堆区。所以在使用Block属性时使用copy修饰，但是ARC中的Block都会在堆上的，系统会默认对Block进行copy操作
- 使用strong也可以，但是block的strong行为默认是用copy的行为实现的，因为block变量默认是声明为栈变量的，为了能够在block的声明域外使用，所以要把block拷贝（copy）到堆，所以说为了block属性声明和实际的操作一致，最好声明为copy。