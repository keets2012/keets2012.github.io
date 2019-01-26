---
title: 设计模式之解释器模式
categories: 设计模式
tags:
  - 设计模式
abbrlink: 55312
date: 2017-03-20 00:00:00
---
属于行为型模式。
## 解释器模式的定义

> Given a language, define a representation for its grammar along with an interpreter that uses the representation to interpret sentences in the language.

给定一种语言，定义他的文法的一种表示，并定义一个解释器，该解释器使用该表示来解释语言中句子。
## 解释器模式的结构

![expression](../../../../pic/expression.png "解释器模式类图")
主要的构成角色如下：

- AbstractExpression：抽象解释器   
声明一个抽象的解释操作，这个接口为抽象语法树中所有的节点所共享。

- TerminalExpression：终结解释器   
实现与文法中的终结符相关联的解释操作。一个句子中的每一个终结符需要该类的一个实例。

- NonterminalExpression：非终结解释器   
对文法中的规则的解释操作。

- Context：环境角色   
包含解释器之外的一些全局信息。

- Client：客户端   
构建表示该语法定义的语言中一个特定的句子的抽象语法树，并调用解释操作。
## 解释器模式的实现
解释波兰表达式（Polish Notation）

- 中缀表达式   
  
  中缀表达式中，二元运算符总是置于与之相关的两个运算对象之间，根据运算符间的优先关系来确定运算的次序，同时考虑括号规则。
  比如： `2 + 5 * (1 - 4)`
  
- 前缀表达式   
  
  波兰逻辑学家 J.Lukasiewicz 于 1929 年提出了一种不需要括号的表示法，将运算符写在运算对象之前，也就是前缀表达式，即波兰式（Polish Notation, PN）。
  比如：`2 + 5 * (1 - 4)` 这个表达式的前缀表达式为 `+ 2 * 5 - 1 4`。
  
- 后缀表达式   
  
后缀表达式也称为逆波兰式（Reverse Polish Notation, RPN），和前缀表达式相反，是将运算符号放置于运算对象之后。
比如：`2 + 5 * (1 - 4)` 用逆波兰式来表示则是：`2 5 1 4 - * +`。

下面我们实现后缀表达式。

- Expression   
抽象表达式类

```java
public interface Expression {
    int interpret(Map<String, Expression> variables);
}
```
- TerminalExpression   
末端表达式： 数字与变量

```java
public class Number implements Expression {
   private int number;

   public Number(int number) {
       this.number = number;
   }

   public int interpret(Map<String, Expression> variables) {
       return number;
   }
}
```

```java
public class Variable implements Expression {
    private String name;

    public Variable(String name) {
        this.name = name;
    }

    public int interpret(Map<String, Expression> variables) {
        if (null == variables.get(name))
            return 0; // Either return new Number(0).
        return variables.get(name).interpret(variables);
    }
}
```

- NonterminalExpression   
非末端表达式： `+ -`操作

```java
public class Minus implements Expression {
    Expression leftOperand;
    Expression rightOperand;

    public Minus(Expression left, Expression right) {
        leftOperand = left;
        rightOperand = right;
    }

    public int interpret(Map<String, Expression> variables) {
        return leftOperand.interpret(variables) - rightOperand.interpret(variables);
    }
}
```

```java
public class Plus implements Expression {
    Expression leftOperand;
    Expression rightOperand;

    public Plus(Expression left, Expression right) {
        leftOperand = left;
        rightOperand = right;
    }

    public int interpret(Map<String, Expression> variables) {
        return leftOperand.interpret(variables) + rightOperand.interpret(variables);
    }
}
```

```java
public class Evaluator implements Expression {
    private Expression syntaxTree;

    public Evaluator(String expression) {
        Stack<Expression> expressionStack = new Stack<Expression>();
        for (String token : expression.split(" ")) {
            if (token.equals("+")) {
                Expression subExpression = new Plus(expressionStack.pop(), expressionStack.pop());
                expressionStack.push(subExpression);
            } else if (token.equals("-")) {
                // it's necessary remove first the right operand from the stack
                Expression right = expressionStack.pop();
                // ..and after the left one
                Expression left = expressionStack.pop();
                Expression subExpression = new Minus(left, right);
                expressionStack.push(subExpression);
            } else
                expressionStack.push(new Variable(token));
        }
        syntaxTree = expressionStack.pop();
    }

    public int interpret(Map<String, Expression> context) {
        return syntaxTree.interpret(context);
    }
}
```

- 客户端

```java
public class InterpreterExample {
    public static void main(final String[] args) {
        final String expression = "w x z - +";
        final Evaluator sentence = new Evaluator(expression);
        final Map<String, Expression> variables = new HashMap<String, Expression>();
        variables.put("w", new Number(5));
        variables.put("x", new Number(10));
        variables.put("z", new Number(42));
        final int result = sentence.interpret(variables);
        System.out.println(result);
    }
}
```

Context也是在客户端定义，定义表达式，装配到解释器中，将Context作为参数传递进去，最后解释器输出结果。

## 总结
本文主要讲解了解释器模式，解释器是一个简单的语法分析工具，它最显著的优点就是扩展性，修改语法规则只需要修改相应的非终结符就可以了，若扩展语法，只需要增加非终结符类就可以了。   
但是，解释器模式会引起类的膨胀，每个语法都需要产生一个非终结符表达式，语法规则比较复杂时，就可能产生大量的类文件，为维护带来非常多的麻烦。
同时，由于采用递归调用方法，每个非终结符表达式只关心与自己相关的表达式，每个表达式需要知道最终的结果，必须通过递归方式，无论是面向对象的语言还是面向过程的语言，递归都是一个不推荐的方式。
由于使用了大量的循环和递归，效率是一个不容忽视的问题。特别是用于解释一个解析复杂、冗长的语法时，效率是难以忍受的。

### 参考
1. [23种设计模式（14）：解释器模式](http://blog.csdn.net/zhengzhb/article/details/7666020)
2. [wikipedia-Interpreter_pattern](https://en.wikipedia.org/wiki/Interpreter_pattern#Java)
