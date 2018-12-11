---
layout:     post                    # 使用的布局（不需要改）
title:      Code Clone Detection Motivating Examples  # 标题 
subtitle:   解释一个新问题时，一个有代表性的小例子很重要 #副标题
date:       2018-12-11              # 时间
author:     Terry Wang                      # 作者
header-img: img/post-bg-2015.jpg    #这篇文章标题背景图片
catalog: true                       # 是否归档
tags:                               #标签
    - 程序设计语言
    - codenet
---

# Motivating Examples for Code Clone Detection

Hi, 有几周没更新是因为去深圳参加了今年的[NASAC会议](http://nasac2018.szu.edu.cn/)，而且又赶上一个小比赛，以及开题、论文等事情。

会议收获颇多，下次单独写篇博客分享；比赛还在进行中，目前有一个作品是[cloud studio的TODO插件](https://studio.dev.tencent.com/plugins/detail/716)，欢迎试用和反馈哦。开题仍在准备中，大方向就是近期经常发blog的code clone detection(CCD)啦。论文实验做了一半，小论文也在准备中。

今天在读[Oreo的论文](https://dl.acm.org/citation.cfm?doid=3236024.3236026)[1]时，分析了下他给出的motivating example，感觉不同类型的克隆不同实际应用中出现的概率是不同的，而且不同应用对这几类克隆的检测要求也不一样。下面就来总结一下作者给出的几类克隆和它们的例子，还有我认为的它们在各类应用中的需求情况。

## 克隆分类 & examples

近几年有个相对公认的分类体系，我在这个系列之前的博客[Code Clone Detection Survey](https://helenawang.github.io/2018/11/11/%E6%8A%80%E6%9C%AF-code-project-Code-Clone-Detection-Survey/)中讲过，这里简单再结合Oreo中给出的例子详细说明一下。

这里先给出Oreo论文中列出的克隆代码的例子。
原始代码如下，实现的功能是给定一个区间，返回区间内的整数序列：

```java
// 原始代码
String sequence(int start, int stop) {
    StringBuilder builder = new StringBuilder();
    int i = start;
    while (i <= stop) {
        if (i > start) builder.append(',');
        builder.append(i);
        i++;
    }
    return builder.toString();
}
```

### Type-1(T1)

完全一致或只修改了空白符、布局和注释。
>Identical code fragments, except for diﬀer-ences in white-space, layout and comments.
下面这个例子是我自己写的。

```java
String sequence(int start, int stop)
{ // 这里增加了注释
    StringBuilder builder = new StringBuilder();
    int i = start;
    while (i <= stop)
    {
        if (i > start)
            builder.append(',');
        builder.append(i);
        i++;
    }
    return builder.toString();
}
```

### Type-2(T2)

修改了标识符名、字面值
>Identical code fragments, except for diﬀer-ences in identiﬁer names and literal values, in addition toType-1 clone diﬀerences.

```java
// Type-2 clone
// differs only in identifiers 这个可以通过转化为token检测出来
String sequence_2(int begin, int end) {
    StringBuilder builder = new StringBuilder();
    int n = begin;
    while (n <= end) {
        if (n > begin) builder.append(',');
        builder.append(n);
        n++;
    }
    return builder.toString();
}
```

前两类克隆，定义很清晰，特征也很明显，基本把源代码转成token流再比对就可以检测出来了。

第三和第四类克隆，尤其是细分后的四类Type-3到Type-4克隆，定义就比较模糊了，下面举Oreo中的例子说明。

### Type-3(T3)

对语句进行了增加、删除、改动。
>Syntactically similar code fragments thatdiﬀer at the statement level. The fragments have statements added, modiﬁed and/or removed with respect to each other,in addition to Type-1 and Type-2 clone diﬀerences.

有些论文对Type-3的克隆做了细分，比如[BigCloneBench](https://ieeexplore.ieee.org/document/6976121)[2]把Type-3到Type-4之间的克隆重新划分为了四类。

#### Very Strongly Type-3

```java
// Very strongly Type-3 clone
// use for-loop instead of while-loop 这个可以通过控制流图检测出来
// 这里越“strong”表示越容易被检测出来
String sequence_3_vs(short start, short stop) {
    StringBuilder builder = new StringBuilder();
    for (short i = start; i <= stop; i++) {
        if (i > start) builder.append('s');
        builder.append(i);
    }
    return builder.toString();
}
```

#### Strongly Type-3

这个没有举例子

#### Moderately Type-3

中等强度的第三类克隆

```java
 // Moderately Type-3 clone 我认为从这类开始，是一个分界，比较难检测，需要的领域可能也没那么多了。
// 这类的token流、控制流图都不一样了，甚至调用的函数都不一样了。
// 不过凭直觉这看上去也不像克隆的了。如果用在学生作业抄袭中，到3_vs这种足够了。
// 如果应用在相同功能的不同实现的查找上，下面这几类还有用。
String sequence_3_m(int start, int stop) {
    String sep = ",";
    String result = Integer.toString(start);
    for (int i = start + 1; ; i++) {
        if (i > stop) break;
        result = String.join(sep, result, Integer.toString(i));
    }
    return result;
}
```

#### Weakly Type-3, which merges with Type-4

这里把弱第三类克隆和第四类克隆合并了，但分别给出了一个例子。

注意，从weakly type-3 克隆开始，就是Oreo的作者提出的Twilight Zone了。

```java
// 以下为twilight zone的克隆
  // Weakly Type-3 clone
  // semantic changes(like variable type, function signature)
  String sequence_3_w(int begin, int end, String sep) {
      String result = Integer.toString(begin);
      for (int n = begin + 1; ; n++) {
          if (end < n) break;
          result = String.join(sep, result, Integer.toString(n));
      }
      return result;
  }
```

### Type-4(T4)

语法结构上不相似，实现的功能一致。
>Syntactically dissimilar code fragments that implement the same functionality

```java
// Type-4 clone
// 这个是递归实现的
static String sequence_4(short n, short m) {
    if (n == m)
        return Short.toString(n);
    return Short.toString(n) + "," + sequence_4((short)(n + 1), m);
}
```

以上的Type-1到Very Strongly Type-3，现有的工具都已经解决得不错了（如SoucererCC[3]），可以说第一到第三类克隆的检测是个基本解决了的问题。所以Oreo这个工具是为了解决当前还未能很好解决的Moderately Type-3 到 Type-4的克隆，并为这类克隆起了一个名字：Twilight Zone。具体Oreo是如何实现的这里先不讲，不过大体也是基于token的方法。

### 两条分界线

我观察Oreo这篇论文中的motivating example后，个人为这个分类体系划出了两条分界线。

- 第一条：Moderately Type-3及以上属于下一类。

    我觉得人眼（就是人看第一眼时的观感，不加任何分析）对前两类和明显的第三类克隆基本能够识别，同样，算法现在也能比较准确地识别了。
    而中等第三类以上的克隆，在一些应用中就可以被认为不是克隆了（比如学生作业抄袭）。因为看上去已经是另一种写法了，学生如果能把别人的代码改动这么大还保证功能正确，也算水平可以了。

- 第二条：Weakly Type-3及以上属于下一类。
    这是Oreo给出的分界线，它把这类及以上的成为Twilight Zone，也是它的工作所关注的克隆类型。
    我觉得这类及以上基本就是“同一功能不同实现”的克隆了，在作业抄袭上这已经不算抄袭了；但在一些其他领域，比如病毒查杀、代码搜索等，应该可以大有用途。

另外，最近几个月读到的很多近几年的clone detection的顶会顶刊论文都出自这个团队之手，，如 BigCloneBench, SoucererCC, Oreo, CCAligner[4]（这篇是与中科大合作的），从2014年开始每年都有新成果，值得持续关注。

## 参考资料

[1] Vaibhav Saini, Farima Farmahinifarahani, Yadong Lu, Pierre Baldi, and Cristina V. Lopes. 2018. Oreo: detection of clones in the twilight zone. In Proceedings of the 2018 26th ACM Joint Meeting on European Software Engineering Conference and Symposium on the Foundations of Software Engineering (ESEC/FSE 2018). ACM, New York, NY, USA, 354-365. DOI: https://doi.org/10.1145/3236024.3236026  
[2] J. Svajlenko, J. F. Islam, I. Keivanloo, C. K. Roy and M. M. Mia, "Towards a Big Data Curated Benchmark of Inter-project Code Clones," 2014 IEEE International Conference on Software Maintenance and Evolution, Victoria, BC, 2014, pp. 476-480.
doi: 10.1109/ICSME.2014.77  
[3] H. Sajnani, V. Saini, J. Svajlenko, C. K. Roy and C. V. Lopes, "SourcererCC: Scaling Code Clone Detection to Big-Code," 2016 IEEE/ACM 38th International Conference on Software Engineering (ICSE), Austin, TX, 2016, pp. 1157-1168.
doi: 10.1145/2884781.2884877  
[4]P. Wang, J. Svajlenko, Y. Wu, Y. Xu and C. K. Roy, "CCAligner: A Token Based Large-Gap Clone Detector," 2018 IEEE/ACM 40th International Conference on Software Engineering (ICSE), Gothenburg, 2018, pp. 1066-1077.  
