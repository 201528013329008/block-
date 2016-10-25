# block-
Block全面分析
 转自https://my.oschina.net/leejan97/blog/268536
 
 摘要: 学习Block从迷惑，到略懂，从理解到顿悟，在此与大家分享。
本文翻译自苹果的文档，有删减，也有添加自己的理解部分。

如果有Block语法不懂的，可以参考fuckingblocksyntax，里面对于Block

为了方便对比，下面的代码我假设是写在ViewController子类中的

1、第一部分

定义和使用Block，

- (void)viewDidLoad
{
    [super viewDidLoad];
    //（1）定义无参无返回值的Block
    void (^printBlock)() = ^(){
        printf("no number");
    };
    printBlock();
    

    printBlock(9);
    
    int mutiplier = 7;
    //（3）定义名为myBlock的代码块，返回值类型为int
    int (^myBlock)(int) = ^(int num){
        return num*mutiplier;
    }
    //使用定义的myBlock
    int newMutiplier = myBlock(3);
    printf("newMutiplier is %d",myBlock(3));
}
//定义在-viewDidLoad方法外部
//（2）定义一个有参数，没有返回值的Block
void (^printNumBlock)(int) = ^(int num){
    printf("int number is %d",num);
};
定义Block变量，就相当于定义了一个函数。但是区别也很明显，因为函数肯定是在-viewDidLoad方法外面定义，而Block变量定义在了viewDidLoad方法内部。当然，我们也可以把Block定义在-viewDidLoad方法外部，例如上面的代码块printNumBlock的定义，就在-viewDidLoad外面。

再来看看上面代码运行的顺序问题，以第（3）个myBlock距离来说，在定义的地方，并不会执行Block{}内部的代码，而在myBlock(3)调用之后才会执行其中的代码，这跟函数的理解其实差不多，就是只要在调用Block（函数）的时候才会执行Block体内（函数体内）的代码。所以上面的简单代码示例，我可以作出如下的结论，

（1）在类中，定义一个Block变量，就像定义一个函数；

（2）Block可以定义在方法内部，也可以定义在方法外部；

（3）只有调用Block时候，才会执行其{}体内的代码；

（PS：关于第（2）条，定义在方法外部的Block，其实就是文件级别的全局变量）

那么在类中定义一个Block，特别是在-viewDidLoad方法体内定义一个Block到底有什么意义呢？我表示这时候只把它当做私有函数就可以了。我之前说过，Block其实就相当于代理，那么这时候我该怎样将其与代理类比以了解呢。这时候我可以这样说：本类中的Block就相当于类自己服从某个协议，然后让自己代理自己去做某个事情。很拗口吧？看看下面的代码，

//定义一个协议
@protocol ViewControllerDelegate<NSObject>
- (void)selfDelegateMethod;
@end

//本类实现这个协议ViewControllerDelegate
@interface ViewController ()<ViewControllerDelegate>
@property (nonatomic, assign) id<ViewControllerDelegate> delegate;

@end
接着在-viewDidLoad中的代码如下，

- (void)viewDidLoad
{
    [super viewDidLoad];
    // Do any additional setup after loading the view from its nib.
    self.delegate = self;
    if (self.delegate && [self.delegate respondsToSelector:@selector(selfDelegateMethod)]) {
        [self.delegate selfDelegateMethod];
    }
}

#pragma mark - ViewControllerDelegate method
//实现协议中的方法
- (void)selfDelegateMethod
{
    NSLog(@"自己委托自己实现的方法");
}
看出这种写法的奇葩地方了吗？自己委托自己去实现某个方法，而不是委托别的类去实现某个方法。本类中定义的一个Block其实就是闲的蛋疼，委托自己去字做某件事情，实际的意义不大，所以你很少看见别人的代码直接在类中定义Block然后使用的，Block很多的用处是跨越两个类来使用的，比如作为property属性或者作为方法的参数，这样就能跨越两个类了。

2、第二部分

__block关键字的使用

在Block的{}体内，是不可以对外面的变量进行更改的，比如下面的语句，

- (void)viewDidLoad
{
    //将Block定义在方法内部
    int x = 100;
    void (^sumXAndYBlock)(int) = ^(int y){
    x = x+y;
    printf("new x value is %d",x);
    };
    sumXAndYBlock(50);
}
这段代码有什么问题呢，Xcode会提示x变量错误信息：Variable is not assigning (missing __block type)，这时候给int x = 100;语句前面加上__block关键字即可，如下，

__block int x = 100;
这样在Block的{}体内，就可以修改外部变量了。

3、第三部分：Block作为property属性实现页面之间传值

需求：在ViewController中，点击Button，push到下一个页面NextViewController，在NextViewController的输入框TextField中输入一串字符，返回的时候，在ViewController的Label上面显示文字内容，

（1）第一种方法：首先看看通过“协议/代理”是怎么实现两个页面之间传值的吧，

//NextViewController是push进入的第二个页面
//NextViewController.h 文件
//定义一个协议，前一个页面ViewController要服从该协议，并且实现协议中的方法
@protocol NextViewControllerDelegate <NSObject>
- (void)passTextValue:(NSString *)tfText;
@end

@interface NextViewController : UIViewController
@property (nonatomic, assign) id<NextViewControllerDelegate> delegate;

@end

//NextViewController.m 文件
//点击Button返回前一个ViewController页面
- (IBAction)popBtnClicked:(id)sender {
    if (self.delegate && [self.delegate respondsToSelector:@selector(passTextValue:)]) {
        //self.inputTF是该页面中的TextField输入框
        [self.delegate passTextValue:self.inputTF.text];
    }
    [self.navigationController popViewControllerAnimated:YES];
}
接下来我们在看看ViewController文件中的内容，

//ViewController.m 文件
@interface ViewController ()<NextViewControllerDelegate>
@property (strong, nonatomic) IBOutlet UILabel *nextVCInfoLabel;

@end
//点击Button进入下一个NextViewController页面
- (IBAction)btnClicked:(id)sender
{
    NextViewController *nextVC = [[NextViewController alloc] initWithNibName:@"NextViewController" bundle:nil];
    nextVC.delegate = self;//设置代理
    [self.navigationController pushViewController:nextVC animated:YES];
}

//实现协议NextViewControllerDelegate中的方法
#pragma mark - NextViewControllerDelegate method
- (void)passTextValue:(NSString *)tfText
{
    //self.nextVCInfoLabel是显示NextViewController传递过来的字符串Label对象
    self.nextVCInfoLabel.text = tfText;
}
这是通过“协议/代理”来实现的两个页面之间传值的方式。

（2）第二种方法：使用Block作为property，实现两个页面之间传值，

先看看NextViewController文件中的内容，

//NextViewController.h 文件
@interface NextViewController : UIViewController
@property (nonatomic, copy) void (^NextViewControllerBlock)(NSString *tfText);

@end
//NextViewContorller.m 文件
- (IBAction)popBtnClicked:(id)sender {
    if (self.NextViewControllerBlock) {
        self.NextViewControllerBlock(self.inputTF.text);
    }
    [self.navigationController popViewControllerAnimated:YES];
}
再来看看ViewController文件中的内容，

- (IBAction)btnClicked:(id)sender
{
    NextViewController *nextVC = [[NextViewController alloc] initWithNibName:@"NextViewController" bundle:nil];
    nextVC.NextViewControllerBlock = ^(NSString *tfText){
        [self resetLabel:tfText];
    };
    [self.navigationController pushViewController:nextVC animated:YES];
}
#pragma mark - NextViewControllerBlock method
- (void)resetLabel:(NSString *)textStr
{
    self.nextVCInfoLabel.text = textStr;
}
好了就这么多代码，可以使用Block来实现两个页面之间传值的目的，实际上就是取代了Delegate的功能。

另外，博客中的代码Sample Code可以再Github下载，如果因为Github被墙了，可以在终端使用git clone + 完整链接，即可克隆项目到本地。

Github中的代码，可以开启两种调试模式，你需要在项目的配置文件BlockSamp-Prefix.pch中注释或者解注释下面的代码，

#define Debug_BlcokPassValueEnable
即可开启两种调试的方式，如果注释了上面的语句就是使用Delegate进行调试；否则使用Block进行调试。
