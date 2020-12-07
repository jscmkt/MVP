# MVP
[简书地址](https://www.jianshu.com/p/19f5aa0435b1)

[001](#mvp-进阶)

#### MVC
MVC 即 Model View Controller（模型 视图 控制器）
MVC 的几个明显的特征和体现：
1. View 上面显示什么东西，取决于 Model。
2. 只要 Model 数据改了，View 的显示状态会跟着更改。
3. Control 负责初始化 Model，并将 Model 传递给 View 去解析展示。
![MVC.png](https://upload-images.jianshu.io/upload_images/2414707-b527b89583243789.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

在传统的 app 中模型数据一般都很简单，不涉及到复杂的业务数据逻辑处理，客户端开发受限于它自身运行的的平台终端，这一点注定使移动端不像 PC 前端那样能够处理大量的复杂的业务场景。然而随着移动平台的各种深入，我们不得不考虑这个问题。传统的 Model 数据大多来源于网络数据，拿到网络数据后客户端要做的事情就是将数据直接按照顺序画在界面上。随着业务的越来越来的深入，我们依赖的 service 服务可能在大多时间无法第一时间满足客户端需要的数据需求，移动端愈发的要自行处理一部分逻辑计算操作。这个时间一惯的做法是在控制器中处理，最终导致了控制器成了垃圾箱，越来越不可维护。

##### MVC案例分析
我们的app都是通过model来更新UI。
```
    cell.model = self.dataArray[indexPath.row];
```
我们平时都是这样对cell进行赋值然后更新UI。这就造成了view对model的依赖。
```
- (void)setNum:(int)num{
    _num                = num;
    self.numLabel.text  = [NSString stringWithFormat:@"%d",self.num];
    self.model.num      = self.numLabel.text; // 指手画脚
```
当界面是数据发生改变的时候，我们的model也必须跟着进行修改。架构的本质是高内聚，低耦合。view应该只负责视图的展示。这里就形成了耦合。这个时候我们就需要view的解耦和Controller的解重。
VC具备调控和管理的能力，不应该做业务具体的执行者。VC的任务就只要简历依赖关系。

#### VC减重
##### 数据提供层
首先是数据请求。新建个类`present`，处理数据请求
```
*************present************
- (instancetype)init{
    if (self = [super init]) {
        [self loadData];
    }
    return self;
}

***********VC************
@property (nonatomic, strong) Present           *pt;
// 数据提供层
self.pt = [[Present alloc] init];
```

##### 数据代理层
把tableview的代理分离出去，建立一个`DataSource`类。
```
*************DataSource************
typedef void (^CellConfigureBefore)(id cell, id model, NSIndexPath * indexPath);

- (id)initWithIdentifier:(NSString *)identifier configureBlock:(CellConfigureBefore)before {
    if(self = [super init]) {
        _cellIdentifier = identifier;
        _cellConfigureBefore = [before copy];
    }
    return self;
}

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath
{
    UITableViewCell *cell = [tableView dequeueReusableCellWithIdentifier:self.cellIdentifier forIndexPath:indexPath];
    id model = [self modelsAtIndexPath:indexPath];
    if(self.cellConfigureBefore) {
        self.cellConfigureBefore(cell, model,indexPath);
    }
    
    return cell;
}


*************VC************
@property (nonatomic, strong) LMDataSource      *dataSource;

    __weak typeof(self) weakSelf = self;
    self.dataSource = [[LMDataSource alloc] initWithIdentifier:reuserId configureBlock:^(MVCTableViewCell *cell, Model *model, NSIndexPath *indexPath) {
        __strong __typeof(weakSelf)strongSelf = weakSelf;
        cell.numLabel.text  = model.num;
        cell.nameLabel.text = model.name;
    }];
    self.tableView.dataSource = self.dataSource;
    //建立关系
    [self.dataSource addDataArray:self.pt.dataArray];

```
对数据代理层提供数据引入口，但是不可以对其进行修改。
cell的数据问题和model的通讯通过block和代理解决。这就是面相协议编程(MVP)。
### MVP
MVP即Model View Presenter（模型 视图 协调器），MVP实现了Cocoa的MVC的愿景。MVP的协调器Presenter并没有对ViewController的声明周期做任何改变，因此View可以很容易的被模拟出来。在Presenter中根本没有和布局有关的代码，但是它却负责更新View的数据和状态。

在`present`中声明个协议`PresentDelegate`
```

*************Cell************
@protocol PresentDelegate <NSObject>
// 需求: UI num -> model
- (void)didClickNum:(NSString *)num indexpath:(NSIndexPath *)indexpath;
- (void)reloadUI;
@end
```
ui中num的改变来告知model。
```
*************Present************
@property (nonatomic,weak) id<PresentDelegate> delegate;

- (void)setNum:(int)num{
    _num                = num;
    self.numLabel.text  = [NSString stringWithFormat:@"%d",self.num];
//    self.model.num      = self.numLabel.text; // 指手画脚
    // 发出响应 model delegate UI 
    if (self.delegate && [self.delegate respondsToSelector:@selector(didClickNum:indexpath:)]) {
        [self.delegate didClickNum:self.numLabel.text indexpath:self.indexPath];
    }
    
}
*************VC************

    __weak typeof(self) weakSelf = self;
    self.dataSource = [[LMDataSource alloc] initWithIdentifier:reuserId configureBlock:^(MVCTableViewCell *cell, Model *model, NSIndexPath *indexPath) {
        __strong __typeof(weakSelf)strongSelf = weakSelf;
        cell.numLabel.text  = model.num;
        cell.nameLabel.text = model.name;
//将cell 的代理 交给数据
        cell.delegate       = strongSelf.pt;
        cell.indexPath      = indexPath;
    }];
```
最后在控制器中更新UI
```
*************Present************
#pragma mark -PresentDelegate
- (void)didClickNum:(NSString *)num indexpath:(NSIndexPath *)indexpath{
        Model *model = self.dataArray[indexpath.row];
        model.num    = num;
            // model - delegate -> UI
            if (self.delegate && [self.delegate respondsToSelector:@selector(reloadUI)]) {
                [self.delegate reloadUI];
            }
}

*************VC************
- (void)reloadUI{
    [self.dataSource addDataArray:self.pt.dataArray];
    [self.tableView reloadData];
}
```
以上就是MVP的简单使用。
但是这只是cell页面比较少，多了会更麻烦。协议胶水代码非常多。
<A HREF="#jinjie"> </A>
### mvp 进阶
我们可以将DataSource 封装起来.界面使用的时候实现需要单独处理的方法
```

- (UITableViewCell *)tableView:(UITableView *)tableView cellForRowAtIndexPath:(NSIndexPath *)indexPath {
    
    KCChannelProfile* liveModel = self.getAdapterArray[indexPath.row];
    
    UITableViewCell *cell = nil;
    
    CCSuppressPerformSelectorLeakWarning (
                                          cell = [self performSelector:NSSelectorFromString([NSString stringWithFormat:@"tableView:cellForKCChannelProfile:"]) withObject:tableView withObject:liveModel];
                                          );
    return cell;
}
//将任务下发出去
- (UITableViewCell *)tableView:(UITableView *)tableView cellForKCChannelProfile:(id)model {
    NSString *cellIdentifier = NSStringFromSelector(_cmd);
    KCHomeTableViewCell *cell = (KCHomeTableViewCell *)[tableView dequeueReusableCellWithIdentifier:cellIdentifier];
    if (!cell) {
        cell = [[KCHomeTableViewCell alloc] initWithStyle:UITableViewCellStyleDefault reuseIdentifier:cellIdentifier];
    }
    
    KCChannelProfile* liveModel = model;
    [cell setCellContent:liveModel];
    
    return cell;
}
```

### context
首先我们需要规范的命名。在vc初始化的时候就创建所有的类
```

    //presentor
    Class presenterClass = NSClassFromString([NSString stringWithFormat:@"KC%@Presenter", name]);
    if (presenterClass != NULL) { // 缓存
        self.context.presenter = [presenterClass new];
        self.context.presenter.context = self.context;
    }
    
    //interactor
    Class interactorClass = NSClassFromString([NSString stringWithFormat:@"KC%@Interactor", name]);
    if (interactorClass != NULL) {
        self.context.interactor = [interactorClass new];
        self.context.interactor.context = self.context;
    }
    
    //view
    Class viewClass = NSClassFromString([NSString stringWithFormat:@"KC%@View", name]);
    if (viewClass != NULL) {
        self.context.view = [viewClass new];
        self.context.view.context = self.context;
    }
```
通过`context`进行缓存，就双方互相持有
```

    //build relation
    self.context.presenter.view = self.context.view;
    self.context.presenter.baseController = self;
    
    self.context.interactor.baseController = self;
    
    self.context.view.presenter = self.context.presenter;
    self.context.view.interactor = self.context.interactor;
```
`context`保存着所有的对象。
```

@class CDDContext;
@class CDDView;

@interface CDDPresenter : NSObject
@property (nonatomic, weak) UIViewController*           baseController;
@property (nonatomic, weak) CDDView*                    view;
@property (nonatomic, weak) id                          adapter; //for tableview adapter

@end

@interface CDDInteractor : NSObject
@property (nonatomic, weak) UIViewController*           baseController;
@end


@interface CDDView : UIView
@property (nonatomic, weak) CDDPresenter*               presenter;
@property (nonatomic, weak) CDDInteractor*              interactor;
@end



//Context bridges everything automatically, no need to pass it around manually
@interface CDDContext : NSObject

@property (nonatomic, strong) CDDPresenter*           presenter;
@property (nonatomic, strong) CDDInteractor*          interactor;
@property (nonatomic, strong) CDDView*                view; //view holds strong reference back to context

@end

```
然后将`context`添加到`NSObject`中.
```

@class CDDContext;

@interface NSObject (CDD)

@property (nonatomic, strong) CDDContext* context;
@end

- (void)setContext:(CDDContext*)object {
    objc_setAssociatedObject(self, @selector(context), object, OBJC_ASSOCIATION_ASSIGN);
}

- (CDDContext*)context {
    id curContext = objc_getAssociatedObject(self, @selector(context));
    if (curContext == nil && [self isKindOfClass:[UIView class]]) {
        
        //try get from superview, lazy get
        UIView* view = (UIView*)self;
        
        UIView* sprView = view.superview;
        while (sprView != nil) {
            if (sprView.context != nil) {
                curContext = sprView.context;
                break;
            }
            sprView = sprView.superview;
        }
        
        if (curContext != nil) {
            [self setContext:curContext];
        }
    }
    
    return curContext;
}

```
值得注意的
1. 上级对下级是强持有，下级对上级是弱引用的，没有问题，但是`view`对`model`直接会发生循环引用，内存无法释放，但是他们都是`context`持有的，`context`对`VC`是弱引用，只要`VC`释放了，其他的都会释放。
![context下的循环引用.png](https://upload-images.jianshu.io/upload_images/2414707-f85d58a53c771caa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2. 为了防止循环引用`context`是弱引用的
```
    objc_setAssociatedObject(self, @selector(context), object, OBJC_ASSOCIATION_ASSIGN);
```
这样除了作用域就会被释放，所以我们还需要给保证他的生命周期
```

@interface KCBaseViewController : ViewController

@property (nonatomic, strong) CDDContext    *rootContext;
@end


    self.rootContext = [[CDDContext alloc] init]; //strong
    self.context = self.rootContext; //weak
```
相当于
```

@property(nonatomic,strong)NSObject *objc;
@property(nonatomic,weak)NSObject *obj;
    self.objc = [[NSObject alloc]init];
    self.obj = self.objc;
```
**子视图的context**
在子视图中，我们没有手动添加，但是我们的`context`是添加在`NSObject`分类中的。
```
    if (curContext == nil && [self isKindOfClass:[UIView class]]) {
        
        //try get from superview, lazy get
        UIView* view = (UIView*)self;
        
        UIView* sprView = view.superview;
        while (sprView != nil) {
            if (sprView.context != nil) {
                curContext = sprView.context;
                break;
            }
            sprView = sprView.superview;
        }
        
        if (curContext != nil) {
            [self setContext:curContext];
        }
    }
```
当子视图没有`context`的时候回遍历查找父视图，知道发现父视图有`context`为止。
**方法触发**
```
[(id<protocol>)(instance) message]
```
