---
title: 控制键盘上右下角按钮的显示(done或是return等)
date: 2016-04-21 18:00:39
tags: iOS
---

UITextField * tf;

tf.returnKeyType = UIReturnKeyDone; // done

其他的类型从该枚举中选择其他的值.

对应的不同类型,该按钮的点击事件是不同的

具体方法写在下面这个代理方法中:

    - ( BOOL )textFieldShouldReturn:( UITextField *)textField {}



如果输入框为textView,情况有一点不同:
    
    
    - (BOOL)textView:(UITextView *)textView shouldChangeTextInRange:(NSRange)range replacementText:(NSString *)text {
    
    //text是最后输入的字符
    
    if ([text isEqualToString:@"\n"]) {
    
    [tv resignFirstResponder];//要做的事..
    
    }
    
    return YES;
    
    }

