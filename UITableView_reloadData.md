reloadData是一个异步的方法，这个调用中包含了异步的操作，当我们reloadData的时候，本意是刷新UITableView，随后会进入一系列的UITableViewDataSource和UITableViewDelegate的回调，其中有些是和reloadData同步发生的，有些则是异步发生的

- 同步：
tableView:numberOfRowsInSection:
- 异步：
tableView:cellForRowAtIndexPath:
tableView:heightForRowAtIndexPath:这个方法有些特殊，在同步和异步的时候都进行了回调。

监听reloadData：
```
dispatch_async(dispatch_get_main_queue(), ^{
    _isReloadDone = NO;
    [tableView reload]; //会自动设置tableView layoutIfNeeded为YES，意味着将会在runloop结束时重绘table
    dispatch_async(dispatch_get_main_queue(),^{
        _isReloadDone = YES;
    });
});
```

```
#import "DBTableView.h"

@interface DBTableView()

@property (nonatomic, copy) void (^reloadDataCompletionBlock)();

@end

@implementation DBTableView

- (void)reloadDataWithCompletion:(void (^)())completionBlock {
    self.reloadDataCompletionBlock = completionBlock;
    [super reloadData];
}

- (void)layoutSubviews {
    [super layoutSubviews];
    if (self.reloadDataCompletionBlock) {
        self.reloadDataCompletionBlock();
        self.reloadDataCompletionBlock = nil;
    }
}

@end

[self.tableView reloadDataWithCompletion:^{
     NSLog(@"完成刷新");
}];
```

没有加入主队列的reload，渲染cell的时机不固定。一般发生跳屏、白屏，就是因为需要等到tableView停下才会继续渲染cell。

reloadSections:withRowAnimation在Document中有一句话是这么解释的:

Calling this method causes the table view to ask its data source for new cells for the specified sections.The table view animates the insertion of new cells in as it animated the old cells out
创建新的sections的新的cell去代替旧的cell。

一般进行异步操作时，当使用reloadRowsAtIndexPaths的时候可能会不生效。

- 1、beginUpdates 和 endUpdates必须成对使用
- 2、使用beginUpdates和endUpdates可以在改变一些行（row）的高度时自带动画，并且不需要Reload row（不用调用cellForRow，仅仅需要调用heightForRow，这样效率最高）。
- 3、在beginUpdates和endUpdates中执行insert,delete,select,reload row时，动画效果更加同步和顺滑，否则动画卡顿且table的属性（如row count）可能会失效。
- 4、在beginUpdates 和 endUpdates中执行 reloadData 方法和直接reloadData一样，没有相应的中间动画。

一般，在添加，删除，选择 tableView中使用，并实现动画效果。

在动画块内，不建议使用reloadData方法，如果使用，会影响动画。

https://developer.apple.com/documentation/uikit/uitableview/1614862-reloaddata

https://www.jianshu.com/p/b482ccaf8bba

https://www.jianshu.com/p/8f566fa95244