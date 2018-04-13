---
title: ViewController那些事儿（三）
date: 2018-04-13 17:59:56
tags:
---

# viewDidLoad何时调用

那VC初始化完成后，viewDidLoad方法何时调用呢？通常我们是点击某个按钮，创建一个VC，然后push。

```
- (IBAction)buttonClicked:(id)sender
{
    TestXibViewController *vc = (TestXibViewController *)[[[NSBundle mainBundle] 		
    loadNibNamed:@"TestXibViewController" owner:nil options:nil] firstObject];
    /** 设置vc需要的参数*/
	[self.navigationController pushViewController:vc animated:YES];
}
```

然后系统就会自动调用viewDidLoad方法了。

如果是通过nib文件初始化的VC（TestXibViewController），那么当awakeFromNib方法调用后，系统会自动调用viewDidLoad。

通过UIStoryboard创建的VC和initWithNibName:bundle:方法创建的VC，其viewDidLoad调用过程类似。那过程是怎么样的呢？

**self.view——>loadViewIfRequired（VC私有方法）——>loadView——>_loadViewFromNibNamed:bundle——>给view赋值——>访问window，设置view大小——>viewDidLoad**

