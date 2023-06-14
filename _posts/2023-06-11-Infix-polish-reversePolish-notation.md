---
layout: post
title: 中缀表达式，波兰式与逆波兰式
date:   2023-06-11 23:10:00 +0800　
categories: Algorithm
tags: C
topping: true
---

### 中缀表达式

我们通常见到的算术表达式就是中缀表达式，如 1+2x3+(4x5-6)x7。中缀表达式可以以树的形式表示为：  

![中缀表达式]({{site.baseurl}}/styles/images/algorithm/stack/infixNotation.png)  

### 波兰式

中缀表达式比较直观，但对计算机来说比较难用于计算。因此在 1920 年，由波兰科学家扬.武卡谢维奇提出了一种不需要括号的计算表达式的方法，将操作符写到操作数之前，称做前缀表达式，又叫波兰式。  

如上述算式 `1+2*3+(4*5-6)*7` 写为波兰式为：`++1*23*-*4567`。  

在计算时，从左到右扫描算式，每遇到相邻两个操作数，则将其与前面的操作符进行计算，用结果替换原来的操作符和操作数，直到计算结束。  

如：  
`++1*23*-*4567`===>`++16*-*4567`===>`+7*-*4567`===>`+7*-'20'67`===>`+7*'14'7`===>`+7'98'`===>105  

### 逆波兰式

逆波兰式是是另一种计算表达式的方法，它与波兰式相反，将操作符写到操作数之后，也称作后缀表达式。  

如上述算式 `1+2*3+(4*5-6)*7` 写为逆波兰式为：`123*+45*6-7*+`。  

逆波兰式可以使用栈来求值，从左到右扫描表达式，每遇到一个操作数则将其入栈，当遇到一个操作符时，将栈顶的两个操作数弹出，并计算结果，然后将结果入栈。当扫描结束时，栈顶元素就是表达式计算结果。  

#### 逆波兰式计算图示

![RPNCalc1.png]({{site.baseurl}}/styles/images/algorithm/stack/RPNCalc1.png)  

![RPNCalc2.png]({{site.baseurl}}/styles/images/algorithm/stack/RPNCalc2.png)  

![RPNCalc3.png]({{site.baseurl}}/styles/images/algorithm/stack/RPNCalc3.png)  

![RPNCalc4.png]({{site.baseurl}}/styles/images/algorithm/stack/RPNCalc4.png)  

#### 逆波兰式求值代码实现

我们使用之前讲到的[抽象数据类型(ADT)-栈]({{site.baseurl}}/2023/06/04/Stack/)栈代码来实现此功能。  

```
#define NOTATION_MAX    (256)

int calcNotationValue(const char* RPNotation)
{
    ttStack* stack = stackInit();

    for (int i = 0; i < strlen(RPNotation); i++) {
        if (isdigit(RPNotation[i]))
            push(stack, RPNotation[i] - '0');
        else {
            int value1 = pop(stack);
            int value2 = pop(stack);
            if (RPNotation[i] == '*')
                value2 *= value1;
            else if (RPNotation[i] == '/')
                value2 /= value1;
            else if (RPNotation[i] == '+')
                value2 += value1;
            else
                value2 -= value1;
            push(stack, value2);
        }
    }
    int total = pop(stack);
    stackDestory(stack);
    return total;
}
```

### 中缀表达式转逆波兰式

中缀表达式转为逆波兰式也可以通过一个栈来实现，先建立一个空栈，然后从左到右扫描原表达式，当遇到一个操作数则直接写入输出表达式，如果遇到一个操作符，如果操作符为左括号“(”，则将左括号入栈，当遇到一个右括号“)”时，从栈中弹出操作符写入输出表达式中，直到弹出左括号为止，左括号不写入表达式。如果是其它操作符，则每次将其与栈顶操作符比较，如果栈顶操作符优先级不低于该操作符，则将栈顶操作符弹出，写入输出表达式，直到栈顶操作符优先级低于该操作符或栈顶为左括号或栈为空为止。  

#### 图示

![infixToRPN1.png]({{site.baseurl}}/styles/images/algorithm/stack/infixToRPN1.png)  

![infixToRPN2.png]({{site.baseurl}}/styles/images/algorithm/stack/infixToRPN2.png)  

![infixToRPN3.png]({{site.baseurl}}/styles/images/algorithm/stack/infixToRPN3.png)  

![infixToRPN4.png]({{site.baseurl}}/styles/images/algorithm/stack/infixToRPN4.png)  

![infixToRPN5.png]({{site.baseurl}}/styles/images/algorithm/stack/infixToRPN5.png)  

![infixToRPN6.png]({{site.baseurl}}/styles/images/algorithm/stack/infixToRPN6.png)  

![infixToRPN7.png]({{site.baseurl}}/styles/images/algorithm/stack/infixToRPN7.png)  

#### 中缀表达式转逆波兰式代码实现

我们同样使用前面讲到的栈来实现此功能。  

```
//infix expression to reverse polish notation
void infixToReversePolish(char* notation)
{
    char infix[NOTATION_MAX] = { 0 };

    strncpy(infix, notation, 255);
    infix[255] = 0;
    memset(notation, 0, NOTATION_MAX);
    int length = 0;
    
    ttStack* stack = stackInit();

    for (int i = 0; i < strlen(infix); i++) {
        if (isdigit(infix[i]))
            notation[length++] = infix[i];
        else if (infix[i] == '(')
            push(stack, infix[i]);
        else if (infix[i] == ')') {
            while (!stackEmpty(stack)) {
                char op = pop(stack);
                if (op == '(')
                    break;
                notation[length++] = op;
            }
        } else {
            while (!stackEmpty(stack)) {
                char op = top(stack);
                if (op == '(' || higherPrecedenceThan(infix[i], op)) 
                    break;
                pop(stack);
                notation[length++] = op;
            }
            push(stack, infix[i]);
        }
    }

    while (!stackEmpty(stack)) 
        notation[length++] = pop(stack);
    notation[length] = 0;

    stackDestory(stack);
}
```

测试代码：  

```
int main(int argc, char* argv[])
{
    char notation[NOTATION_MAX] = { 0 };
    snprintf(notation, NOTATION_MAX - 1, "1+2*3+(4*5-6)*7");
    infixToReversePolish(notation);
    printf("reversePolishNotation:%s\n", notation);
    int total = calcNotationValue(notation);
    printf("total:%d\n", total);

    return 0;
}
```

--- 
**参考**：  
<数据结构与算法分析-C语言实现> P3
