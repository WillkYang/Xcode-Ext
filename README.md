>Xcode8开放了新的一个Extension：Xcode Source Editor Extension，目的是让开发者可以正规的自主为IDE编写插件，虽然说系统现提供的功能还比较拮据，但是不妨碍我们了解和使用，本文主要介绍Xcode Source Editor Extension的功能，并演示一个简单的插件的实现～

###### 一、实现功能
1.删除无用的类头文件，要求类名和文件名一致
2.删除重复导入的头文件，只保留一个

###### 二、编写代码
1.新建项目，然后新建一个Target，类型选择Xcode Source Editor Extension，完成之后设置target的签名和项目的签名一致。
![在New中选择Target](http://upload-images.jianshu.io/upload_images/459563-6bad7552f26e7442.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


2.在info.plist中可以修改插件显示名称Bundle name和其它对Extension的设置。

![插件的info.plist](http://upload-images.jianshu.io/upload_images/459563-d97284d68d580069.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

3.系统默认为我们生成SourceEditorCommand文件，此处我们也可以在info里边修改配置项，类似于项目中系统生成的Main.storyboard。插件的重点基本在：
 ```
- (void)performCommandWithInvocation:(XCSourceEditorCommandInvocation *)invocation completionHandler:(void (^)(NSError * _Nullable nilOrError))completionHandler
```
用户调用我们的插件时，系统会回调这个方法，
######XCSourceEditorCommandInvocation
>Information about the source editor command that the user invoked, such as the identifier of the command, the text buffer on which the command is to operate, and whether the command has been canceled by Xcode or the user.

其中invocation.buffer是编辑器的全部文本
```
/** 当前编辑器的全部文件内容 */
@property (readonly, strong) NSMutableArray <NSString *> *lines;
/** 是当前选中的文本 */
@property (readonly, strong) NSMutableArray <XCSourceTextRange *> *selections;
```
我们在回调方法中编写如下代码：

![源码图片一份，方便查看](http://upload-images.jianshu.io/upload_images/459563-eb2c20a81e60ee46.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

```
    //headerDict存放文本中所有的头文件
    NSMutableDictionary <NSString*, NSNumber *>*headerDict = [NSMutableDictionary dictionary];
    //willCheckDict存放将要删除的头文件
    NSMutableDictionary <NSNumber*, NSString *>*willCheckDict = [NSMutableDictionary dictionary];
    
    //遍历编辑器每一行
    for (int idx = 0; idx < invocation.buffer.lines.count; idx++) {
        
        NSString *lineCode = invocation.buffer.lines[idx];
        
        //若willCheckDict文件不为空，则进行是否使用了该头文件的判断
        if (willCheckDict.count > 0) {
            [willCheckDict enumerateKeysAndObjectsUsingBlock:^(NSNumber * _Nonnull key, NSString * _Nonnull checkString, BOOL * _Nonnull stop) {
                if ([lineCode containsString:checkString]) {
                    if (![lineCode containsString:@"#import"]) {
                        if ([headerDict[checkString] isEqualToNumber: @1]) {
                            //若使用了该头文件，则从willCheckDict字典中提出该项
                            [willCheckDict removeObjectForKey:key];
                            //同时设置该头文件已经检查过，若后续仍出现该头文件，则可以进行删除
                            headerDict[checkString] = @0;
                        }
                    }
                }
            }];

        }
        
        //检测代码是否含有#import为头文件标志；+号我们认为是类扩展的标志
        if ([lineCode containsString:@"#import"] && ![lineCode containsString:@"+"]) {
            //解析获取类名
            NSRange range1 = [lineCode rangeOfString:@"\""];
            NSRange range2 = [lineCode rangeOfString:@"\"" options:NSBackwardsSearch];
            NSRange zeroRange = NSMakeRange(0, 0);
            
            if (!(NSEqualRanges(range1, zeroRange) || NSEqualRanges(range2, zeroRange))) {
                NSRange findRange = NSMakeRange(range1.location + 1, range2.location - range1.location - 3);
                NSString *classString = [lineCode substringWithRange:findRange];
                willCheckDict[@(idx)] = classString;
                headerDict[classString] = @1;
            }
        }
    }
    
    
    //取出需要删除的行
    NSMutableIndexSet *index = [NSMutableIndexSet indexSet];
    [willCheckDict.allKeys enumerateObjectsUsingBlock:^(NSNumber * _Nonnull obj, NSUInteger idx, BOOL * _Nonnull stop) {
        [index addIndex:obj.integerValue];

    }];
    
    //删除不符合条件的行
    [invocation.buffer.lines removeObjectsAtIndexes:index];
    
    //通知系统完成
    completionHandler(nil);
```

##### 三、测试结果

1.运行，选择Xcode8

![command+r运行插件](http://upload-images.jianshu.io/upload_images/459563-34347f0d276c2e55.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

2.可以看见灰色的Xcode实例。随便选择一个项目打开

![测试的Xcode，用于区别正式的Xcode](http://upload-images.jianshu.io/upload_images/459563-5a4e6c5afb7641ca.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

3.测试。测试文件中含有未使用的头文件和冗余的头文件

![处理前代码](http://upload-images.jianshu.io/upload_images/459563-9850d7c6a1352711.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

4.Editor中选择插件运行

![不出意外Editor菜单最底下一栏，此处名字可以在info.plist修改](http://upload-images.jianshu.io/upload_images/459563-d1c732c5ecc548e3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

5.检验运行结果

![处理后代码](http://upload-images.jianshu.io/upload_images/459563-64f96edc2777c200.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

啦啦啦，多余的头文件已经被成功检测到并且移除了了～

##### 四、总结
至此，我们完成并测试通过了一个简单的Xcode插件的编写。主要目的是简单了解和使用Xcode8的插件，如果觉得有用，可以找到product里边的文件复制出来打开，然后在系统设置辅助功能中启用，最后在Xcode中绑定快捷键即可食用。当然，功能十分简陋，还请大神勿怪～

不足：受限于系统现有API，运行插件时，只能获取到当前编辑的文件，无法获取整个项目文件来分析，故很多功能暂时无法实现，如支持更加智能的检测等等，以后系统若能提供项目空间的文件访问和GUI支持，则插件可以发挥更大作用～

知识链：
[WWDC2016](https://developer.apple.com/videos/play/wwdc2016/414/)
[iOS 10 Day By Day: Xcode Source Editor Extensions](http://www.cocoachina.com/ios/20161212/18344.html)
[使用 Xcode Source Editor Extension开发Xcode 8 插件](http://blog.csdn.net/zhouzhoujianquan/article/details/52600763)


欢迎加群讨论其它～：578874451
