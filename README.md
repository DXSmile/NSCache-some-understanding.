# NSCache-some-understanding.
对于NSCache的一些理解
# 对于NSCache的一些理解
****
**对于有一定开发经验的iOS攻城狮来说,我们在对一个APP数据做存储和内存优化的时候,不可避免的需要对缓存做相应的处理,而且缓存处理的优劣,往往也是决定一个APP能否长线发展的重要因素之一,今天就来说一下经常容易被我们忽略的一个苹果官方提供的一套缓存机制--->NSCache**
****

### 一.什么是NSCache?
#### 1. NSCache苹果提供的一套缓存机制
***主要作用于内存缓存的管理方面;***
**在没有引入NSCache之前,我们要管理缓存,都是使用的NSMutableDictionary来管理,如:**
```
// 定义下载操作缓存池
@property (nonatomic, strong) NSMutableDictionary *operationCache;
// 定义图片缓存池
@property (nonatomic, strong) NSMutableDictionary *imageCache;
```

**然而,使用NSMutableDictionary来管理缓存是有些不妥的, 知道多线程操作原理的开发者都明白, NSMutableDictionary在线程方面来说是不安全,这也是苹果官方文档明确说明了的,而如果使用的是NSCache,那就不会出现这些问题.所以接下来我们先看看二者的区别:**
##### &1 NSCache和NSMutableDictionary的相同点与区别:
相同点：
 NSCache和NSMutableDictionary功能用法基本是相同的。
 
区别：

1. NSCache是线程安全的，NSMutableDictionary线程不安全
NSCache线程是安全的，Mutable开发的类一般都是线程不安全的
2. 当内存不足时NSCache会自动释放内存(所以从缓存中取数据的时候总要判断是否为空)
3. NSCache可以指定缓存的限额，当缓存超出限额自动释放内存
 缓存限额：
  1) 缓存数量
  @property NSUInteger countLimit;                 
 2) 缓存成本
 @property NSUInteger totalCostLimit;

4. 苹果给NSCache封装了更多的方法和属性,比NSMutableDictionary的功能要强大很多
	
	
#### 2.代码演示:
**先定义缓存池,并懒加载初始化:**

	#import "ViewController.h"

	@interface ViewController () <NSCacheDelegate>
	
	// 定义缓存池
	@property (nonatomic, strong) NSCache *cache;
	@end

	@implementation ViewController
	- (NSCache *)cache {
    if (_cache == nil) {
        _cache = [[NSCache alloc] init];
        // 缓存中总共可以存储多少条
        _cache.countLimit = 5;
        // 缓存的数据总量为多少
        _cache.totalCostLimit = 1024 * 5;
    }
    return _cache;
	}
	
    - (void)viewDidLoad {
      [super viewDidLoad];
      // Do any additional setup after loading the view, typically from a nib.
    
      //添加缓存数据
       for (int i = 0; i < 10; i++) {
        [self.cache setObject:[NSString stringWithFormat:@"hello %d",i] forKey:[NSString stringWithFormat:@"h%d",i]];
        NSLog(@"添加 %@",[NSString stringWithFormat:@"hello %d",i]);
       }
    
      //输出缓存中的数据
       for (int i = 0; i < 10; i++) {
        NSLog(@"%@",[self.cache objectForKey:[NSString stringWithFormat:@"h%d",i]]);
       }
    
    }

控制台输出结果为:
   ![控制台输出结果](https://static.oschina.net/uploads/img/201602/16215315_WTwK.png "在这里输入图片标题")

**通过输出结果可以看出: **
>1.当我们使用NSCache来创建缓存池的时候,我们可以很灵活的设置缓存的限额, 
>2.当程序中的个数超过我们的限额的时候,会先移除最先创建的
>3.如果已经移除了,那么当我们输出缓存中的数据的时候,就只剩下后面创建的数据了;

#### 3. 演示NSCache的代理方法
先设置代理对象:
         - (void)viewDidLoad {
      [super viewDidLoad];
      // Do any additional setup after loading the view, typically from a nib.
      //设置NSCache的代理
      self.cache.delegate = self;
调用代理方法: 这里我仅用一个方法来演示:
        
		 //当缓存被移除的时候执行
             - (void)cache:(NSCache *)cache willEvictObject:(id)obj{
		    NSLog(@"缓存移除  %@",obj);
           }

输出结果为:
![输出结果](https://static.oschina.net/uploads/img/201602/16221517_w42r.png "在这里输入图片标题")

**通过结果可以看出:**
**NSCache的功能要比NSMutableDictionary的功能要强大很多很多;**


#### 4.当遇到内存警告的时候,
代码演示:
```
- (void)didReceiveMemoryWarning {
    [super didReceiveMemoryWarning];
    //当收到内存警告，清除内存
    [self.cache removeAllObjects];
    //输出缓存中的数据
    for (int i = 0; i < 10; i++) {
        NSLog(@"%@",[self.cache objectForKey:[NSString stringWithFormat:@"h%d",i]]);
    }
}
```
控制台输出结果:
![收到内存警告](https://static.oschina.net/uploads/img/201602/16220314_PPd8.png "在这里输入图片标题")
**通过结果可以看出:**
**当收到内存警告之后,清除数据之后,NSCache缓存池中所有的数据都会为空!**

#### 5.当收到内存警告，调用removeAllObjects 之后，无法再次往缓存池中添加数据

代码演示:

```
- (void)didReceiveMemoryWarning {
    [super didReceiveMemoryWarning];
    //当收到内存警告，调用removeAllObjects 之后，无法再次往缓存中添加数据
    [self.cache removeAllObjects];
    //输出缓存中的数据
    for (int i = 0; i < 10; i++) {
        NSLog(@"%@",[self.cache objectForKey:[NSString stringWithFormat:@"h%d",i]]);
    }
}

// 触摸事件, 以便验证添加数据
- (void)touchesBegan:(NSSet<UITouch *> *)touches withEvent:(UIEvent *)event {
    [self.cache removeAllObjects];
    
    //添加缓存数据
    for (int i = 0; i < 10; i++) {
        [self.cache setObject:[NSString stringWithFormat:@"hello %d",i] forKey:[NSString stringWithFormat:@"h%d",i]];
//        NSLog(@"添加 %@",[NSString stringWithFormat:@"hello %d",i]);
    }
    
    //输出缓存中的数据
    for (int i = 0; i < 10; i++) {
        NSLog(@"%@",[self.cache objectForKey:[NSString stringWithFormat:@"h%d",i]]);
    }

}
```

控制台输出结果为:
![输出结果](https://static.oschina.net/uploads/img/201602/16221914_fFxb.png "在这里输入图片标题")

**通过输出结果,我们可以看出:**
**当收到内存警告，而我们又调用removeAllObjects 之后，则无法再次往缓存中添加数据;**


		

