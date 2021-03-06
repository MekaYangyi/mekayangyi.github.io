# clean code读书笔记




命名，很多人戏称为编程中最难的事情。从实践经验出发来看，一个好的命名能够让代码阅读者迅速的知道代码的实际用途，而坏的命名不仅仅是让人摸不着头脑，而且可能误导他人。

 本文主要是clean code第二章有意义的命名的笔记，该章节系统的讲述了什么才是一个好的名字。

1. 名副其实

   > 如果命名需要注释，则不是一个好的命名。
   >
   > 命名需要表达出准确的含义，不应该使用magic num或者i、j、k这类无意义的变量名。

2. 避免误导

   > 0和O、1、I和l。这类视觉上容易混淆的名字不需要使用。
   >
   > 假设存在一个vector，里面存储这actor_rid，但是你命名为actor_list，导致阅读者误认为是用链表实现的。

3. 做有意义的区分

   > 类似a1、a2、a3不能够带来任何信息的变量命名
   >
   > 类似thexx、xxdata、xxinfo、xxobject这类没有意义的前缀、后缀
   >
   > 不使用匈牙利命名法带上类型信息，现在ide随时可以看到类型
   >
   > 变量的命名需要带有含义，不要带上冗余的无效的信息。

4. 使用读得出来的名称

   > 如果变量难易阅读的话，对阅读者总是负担，难以记住也难易和他人交流

5. 使用可搜索的名称

   > 作用域越大变量名越长，便于搜索同时不会重复。

6. 避免使用编码

   > 匈牙利命名法在现代ide没有必要，同时影响ide的自动补全，修改类型名同时需要修改变量名
   >
   > 类似m_的前缀没有必要

7. 避免翻译

   > 缩写

8. 类名

   > 类名应该是名词或者名词短语，不应该是动词

9. 方法名

   > 方法名应该是动词或者动词短语，类似set_xx、is_xx

10. 别扮可爱

    > 不要使用难以联想到的词，即使你认为很精妙，直接了当的命名。

11. 每个概念对应一个词

    > controller、manager、driver。英语中有很多语义相似的词语，代码中统一使用一个词来描述。

12. 不使用双关

    > 双关代表者二义性。比如add，有可能是insert的意思，有可能是append的意思。

13. 使用解决方案领域名称

    > 计算机科学属于、算法名、模式名。使用领域内专有名词利于有共同经验的人理解程序

14. 使用源自所涉问题领域名称

    > 我们是游戏编程，有很多游戏专有名词。比如exp、level、dps等，更业务的代码使用这些名词易于理解

15. 添加有意义的语境

    > 将相关的变量设置统一的前缀、后缀，建立语境。

16. 不要添加没用的语境

    > 比如给同一个项目里的所有类添加上项目名称。
