---
layout: post
title: 用栈实现平衡符号的检查
date:   2023-06-10 22:30:00 +0800　
categories: Algorithm
tags: C
topping: true
---

### 平衡符号检测

编译器中经常会有测试符号是否平衡的语法检测，即有开符号（'(', '[', '{'），有对应的闭符号（')', ']', '}'）与其按顺序对应。如 “[()]” 是合法的，“[(])” 是非法的。  

符号是否平衡可以通过一个栈来检测，首先创建一个空栈，检查原字符串，当遇到一个开符号时，则将其入栈，当遇到一个闭符号时，将其与栈顶符号进行比较，如果栈为空，则符号不平衡，如果栈顶符号是该闭符号对应的开符号，则弹出该开符号，否则符号不平衡。直到检测完原字符串，如果此时栈不为空，说明符号不平衡，否则符号平衡。  

### 代码实现

我们使用之前讲到的[抽象数据类型(ADT)-栈]({{site.baseurl}}/2023/06/04/Stack)代码来实现此功能。  

```
bool signBalance(const char* string)
{
    const char* openSign = "([{";
    const char* closeSign = ")]}";

    ttStack* stack = stackInit();
    for (int i = 0; i < strlen(string); i++) {
        for (int j = 0; j < strlen(openSign); j++) {
            if (string[i] == openSign[j]) 
                push(stack, string[i]);
        }
        for (int j = 0; j < strlen(closeSign); j++) {
            if (string[i] == closeSign[j]) {
                char pushChar = pop(stack);
                if (pushChar != openSign[j])
                    return false;
            }
        }
    }

    bool isEmpty = stackEmpty(stack);
    stackDestory(stack);
    return isEmpty;
}
```

--- 
