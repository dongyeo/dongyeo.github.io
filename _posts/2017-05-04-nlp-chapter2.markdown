---
layout:     post
title:      "徒手撸个NLP分词器（2）"
date:       2017-05-04 00:00:00 +0800
author:     "DongYeo"
header-img: "img/post-bg-05.jpg"
tags: ["NLP"]
---

 你吃过猪肉，但你会做么？———尼古拉斯·造轮子·董

（注：本系列文章为 [《自然语言处理原理与技术实现》](https://www.amazon.cn/%E5%9B%BE%E4%B9%A6/dp/B01G8JOUSO/ref=sr_1_5?ie=UTF8&qid=1493475144&sr=8-5&keywords=%E8%87%AA%E7%84%B6%E8%AF%AD%E8%A8%80%E5%A4%84%E7%90%86) 学习笔记，点击链接可购买，购买前请看评论哈~）


## 一元分词

### 一元概率模型

通常，一句话分词可能会有多种组合情况，不同的组合情况又会产生不同的意义，这种情况称之为歧义。

例如：“从中学到知识“ 可能结果如下

- 从 /中学 /到/ 知识
- 从中 / 学到 / 知识

显而易见，第二种分词方案更加符合实际的语义。那么如何解决歧义呢？

现在我们假设对于给定的输入串C 有两种分词方案：S1 S2

我们可以根据S1 S2在语料库中的组合概率来评估出哪种分词的更加符合语境。简单来说，就是比较 
P(S1|C)、P(S2|C)的概率。根据bias公式，我们可以得出他们的概率比较可以变成P(S1)以及P(S2)大小。
![bias]({{ $baseurl}}/img/bias.png)

那么，对于一句话C，在一元分词，我们所要做的就是找出一个分词方案，使得P(S)最大。那么P(S)值如何求解呢？

现在，我们假设一句话C=C1C2..Cn ,可能产生的切分词序列S=W1W2...Wm (m<=n),那么我们可以将近似得出以公式（假设词与词之间的出现是没有任何关系的，即词与词之间的信息熵为0）：

![bias]({{ $baseurl}}/img/PS.png)

而P(W)是词出现的概率，这个在语聊库中就是词出现的次数/总词数。

所以，上面的例子，我们可以变成比较下面计算结果的大小：

- P(从)\*P(中学)\*P(到)\*P(知识)
- P(从中)\*P(学到)\*P(知识)

### 切分词图

在讲解一元分词之前，先介绍下一下切分词图的概念，这将有利于我们理解一元分词的计算，并且，这一概念将会贯穿后面几个章节。

我们可以把一段话的切分点看成是一个个的点，切分点之间的词看成是边。这么理解可能有拗口，请看下图：

![word-graph]({{ $baseurl}}/img/word-graph.png)

那么上面的切分方案就有两种：

   - 路径1 0-1-3-4-6
   - 路径2 0-2-4-6
   
前驱词：某一节点前的所有可能的词语称为节点的前驱词，例如上图中的节点4的前驱词为：{到,学到}
   
### 一元概率分词实现

在第一部分我们已经知道，找到一个词的最佳分词方案，就是计算求得各个分词方案的词的概率累乘结果的最大值。
从另外一个角度看，就是计算切分词图的最大权路径，这里使用动态规划的算法来计算：

具体的伪代码如下：

    从一段话的0节点开始遍历
        从当前节点找出所有前驱节词，并遍历
        计算P(前驱词节点)*P(前驱词)//候选节点概率
        如果这个概率大于当前节点的最大概率
            把这个概率当成但钱节点的概率
            把当前词最为当前节点的最佳前驱词

那么最佳的分词结果就是从末尾节点回溯节点的最佳前驱词。

下面是具体的代码

```java
public class UnarySegment implements ISegment {

    private static final double MIN_VALUE = -999999999;

    public UnarySegment(IDictionary dictionary) {
        this.dictionary = dictionary;
    }

    private IDictionary dictionary;

    public List<Word> seg(String sentence) {
        int[] prevNode = new int[sentence.length() + 1];
        double[] prob = new double[sentence.length() + 1];
        Word[] words = new Word[sentence.length() + 1];
        List<Word> prevWords = new ArrayList<Word>();
        List<Word> ret = new ArrayList<Word>();

        for (int i = 1; i <= sentence.length(); i++) {
            double maxProb = MIN_VALUE;
            int maxNode = 0;
            Word maxWord = null;
            dictionary.matchAll(sentence, i - 1, prevWords);

            for (Word word : prevWords) {
                double wordProb = Math.log(word.freq) - Math.log(dictionary.getN());
                int start = i - word.term.length();
                double nodeProb = prob[start] + wordProb;
                if (nodeProb > maxProb) {
                    maxNode = start;
                    maxProb = nodeProb;
                    maxWord = word;
                }
            }
            prob[i] = maxProb;
            prevNode[i] = maxNode;
            words[i] = maxWord;
        }
        Stack<Integer> path = new Stack<Integer>();
        for (int i = sentence.length(); i > 0; ) {
            path.add(i);
            i = prevNode[i];
        }
        while (!path.empty()) {
            ret.add(words[path.pop()]);
        }
        return ret;
    }
}
```




