---
layout:     post
title:      "徒手撸个NLP分词器（1）"
date:       2017-04-29 21:00:00 +0800
author:     "DongYeo"
header-img: "img/post-bg-01.jpg"
tags: ["NLP"]
---

 你吃过猪肉，但你会做么？———尼古拉斯·造轮子·董

## 前言

前一阵子，因为项目需求，使用到了分词工具，需求是在已给的一段话里实现以下需求：
    
   - 根据事先整理好的语料库，用不同颜色的字体，标注出一段话中各个词的频率等级。
   - 根据分词结果，告诉当前语句是否通顺。
   
第一个需求，只要使用分词工具将词切分出来，然后根据频率渲染字的颜色就可以。

第二个需求，语句是否通顺，当时我使用的是将分词结果的词性序列跟语料库中的词性序列概率统计来比较，根据词性序列频率，给出语句通顺程度。

（注：我使用的是开源的NLP工具 [HANLP](https://github.com/hankcs/HanLP)，它是由一系列模型与算法组成的Java工具包，目标是普及自然语言处理在生产环境中的应用。HanLP具备功能完善、性能高效、架构清晰、语料时新、可自定义的特点。）

实现完需求以后，总觉得不是很完美，心中有了两个疑惑：

- 分词是怎么实现的？
- 语句通顺，语法检查到底是不是我上述方法来实现的？是否有更加完美的数学模型来支持？

于是，带着疑问，我开始了自学NLP的旅程。（对了，我自学使用的书 [《自然语言处理原理与技术实现》](https://www.amazon.cn/%E5%9B%BE%E4%B9%A6/dp/B01G8JOUSO/ref=sr_1_5?ie=UTF8&qid=1493475144&sr=8-5&keywords=%E8%87%AA%E7%84%B6%E8%AF%AD%E8%A8%80%E5%A4%84%E7%90%86)，买前请看评论，其他不多说）

直接刚，分词是怎么找出词的。

## 直接刚，分词是怎么找出词的（基本原理）

类似英文这类的语言文字，他们天然的使用空格作为词语词之间的分隔，这对NLP工具来说是非常友好的。但是对于类似于汉语这类的语言，词与词之间，是没有分隔的。
对于这类的语言，NLP先要把词分隔出来，才能做更近一步的处理。

那么问题来了，怎么将连着的一段话中的汉语分隔出来呢？

仔细想想，分词要做的事情就是我们日常阅读中断句工作，以阅读“今天要去上学”为例，基本的流程大概是这样的（假设我们现在是一个词汇量有限的一年级学生）：

1. 今（小学生：“今”这个不是一个词）
2. 今天（小学生：“今天“”这是一个词）
3. 今天\要（小学生：“要”是一个词）
4. 今天\要\去（小学生：“去”也是一个词）
5. 今天\要\去\上（小学生：“上”这不是一个词）
6. 今天\要\去\上学（小学生：“上学”是一个词，哇分词结束了！！！）

以上就是一个最最简单的分词过程，从一句话的开始往下读，去搜索自己的词汇表，如果从上一个断句的点到当前的字无法组成一个词，继续读下一个字，直到这个组成字序列出现在词汇表中，他才算一个词。

所以，汉语分词，首先需要一个词典，以词典为基础，来做分词，本章就来讲讲Java中是如何实现词典。

## 计算机中词典如何实现（数据结构）

通常，计算机要使用的词库，都是以一定格式存储的文本文件或者二进制文件，同他的结构类似于下面这样：

    啊 100
    啊哈 22
    把 100
    ......
  
上面就是一个最简单的词库，每一行一个词，词后面跟空格隔开一个他在本词库中出现的次数。

那么计算机是用什么数据结构存储词典，又能保证词典的快速检索的呢？

不卖关子，分词中使用的是一种叫做Trie三叉树的数据结构。这种树结构和我们大学学的二叉树很像，假设我们有一个简单的词库，有一下词语：{今天,要,去,上学}为例，如下图所示为其存储结构：

![trie-tree]({{ site.baseurl}}/img/trie-tree.png)

词典以树状结构存储，每个节点存储一个字符，在Java中就是一个char。
一个节点将会链接三个节点，左节点、相等节点、右节点。
左（右）节点用实线链接，左边是存储字符比当前节点小的词，右边反之，中间是相等节点。
红色节点代表一个词的结束，这个节点上存储着沿着一个完整的词。（这里，你可能发现存储的词结构是反着来的，这个后面再解释）

这么解释可能有些拗口，我们直接上代码吧：

```java
//词数据结构
public class Word {
    //词
    public String term;
    //词频
    public int freq;
    //其他属性忽略
}
//节点数据结构
public final class TSTNode {
    //词，只有结束节点才不为空
    private Word data;
    //当前节点的字符
    private char spliter;
    //左节点
    private TSTNode loNode;
    //相等节点
    private TSTNode eqNode;
    //右节点
    private TSTNode hiNode;
}
```

基本的数据结构就是上面两个了，那么，我们要如何加载词典呢？

下面是一个词典的数据结构，createTSTNode函数向字典加入一个词，并返回一个当前词的TSTNode指针。
基本的思想就是当词典加载的时候，每个词都调用该方法，该方法从词的末尾开始，跟根节点中存储的字符比较，依次递归，在树中搜索并根据字符的比较，在树中添加节点，直到这个词被存储到这课树中。

```java
public class TernarySearchTrieDic implements IDictionary {
     private TSTNode rootNode;
     public TSTNode createTSTNode(String key) throws NullPointerException, IllegalArgumentException {
         if (key == null) {
             throw new NullPointerException("空指针异常");
         }
         int charIndex = key.length() - 1;
         if (rootNode == null) {
             //根节点为空，直接新建一个节点作为根节点，节点中存储这个词的最后一个字符
             rootNode = new TSTNode(key.charAt(charIndex));
         }
         TSTNode currentNode = rootNode;
         //开始搜索树
         while (true) {
             //字符比较，大于零进入右树，小于零进入左树，等于零证明当前节点字符在该词的路径上
             int comp = (key.charAt(charIndex) - currentNode.getSpliter());
             if (comp == 0) {
                 //相等，比较词的上一个字符
                 charIndex--;
                 if (charIndex <= -1) {
                     //词遍历结束，找到节点
                     return currentNode;
                 }
                 if (currentNode.getEqNode() == null) {
                     //创建当前节点的相等节点，节点存储的字符是词的下一个字符
                     currentNode.setEqNode(new TSTNode(key.charAt(charIndex)));
                 }
                 currentNode = currentNode.getEqNode();
             } else if (comp < 0) {
                 if (currentNode.getLoNode() == null) {
                     currentNode.setLoNode(new TSTNode(key.charAt(charIndex)));
                 }
                 currentNode = currentNode.getLoNode();
             } else {
                 if (currentNode.getHiNode() == null) {
                     currentNode.setHiNode(new TSTNode(key.charAt(charIndex)));
                 }
                 currentNode = currentNode.getHiNode();
             }
         }
     }   
}
```


词典相关的基本的数据结构基本上讲完了，至于怎么分词的，下章继续哈~

另外，五一节快乐哦~




