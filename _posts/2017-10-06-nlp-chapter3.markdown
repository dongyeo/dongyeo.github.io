---
layout:     post
title:      "徒手撸个NLP分词器（3）"
date:       2017-10-06 00:00:00 +0800
author:     "DongYeo"
header-img: "img/post-bg-01.jpg"
tags: ["NLP"]
---

 你吃过猪肉，但你会做么？———尼古拉斯·造轮子·董

（注：本系列文章为 [《自然语言处理原理与技术实现》](https://www.amazon.cn/%E5%9B%BE%E4%B9%A6/dp/B01G8JOUSO/ref=sr_1_5?ie=UTF8&qid=1493475144&sr=8-5&keywords=%E8%87%AA%E7%84%B6%E8%AF%AD%E8%A8%80%E5%A4%84%E7%90%86) 学习笔记，点击链接可购买，购买前请看评论哈~）


## 二元分词

通常一段文字，词语之间都是有一定的关联度的，例如动词后面接名词的可能性比接动词的可能要大等等。
这样我们在计算最佳前驱节点到当前节点的概率的时候，可以考虑再前面一个切分词，以得到更加有区分度的概率。

因此，在计算当前节点概率的时候，我们需要将P(Wi)的概率替换为P(wi\|wi-1)。而这个概率，我们可以使用FREQ(wi,wi-1)/FREQ(wi-1)来近似。
例如：P(意见|有)=FREQ(有，意见)/FREQ(有)。

为了保证数据不因为稀疏导致的0概率出现，我们使用这个公式来近似估计并保证归一：

    P(wi|wi-1) = λ1*(FREQ(wi)/N)+λ2*(FREQ(wi,wi-1)/FREQ(wi-1))
    其中，λ1+λ2 = 1 N=所有词语的出现次数

## 基本算法
二元分词的算法大体上与一元分词近似，只是计算最佳前驱节点的概率公式不同，需要考虑当前节点前驱词的最佳前驱词与当前词的概率，在循环中计算当前节点的最大概率,就是求当前公式的最大值：

      P(StartNode(wx))\*P(wx|BestPrev(StartNode(wx)))
      其中，startNode是当前节点的前驱词的开始节点（Nodei-wx.length）,BestPrev(StartNode(wx))就是开始节点的最佳前驱词。

以“有意见分歧”这句话为例子，将会如下计算：

      P(Node1) = P(有)
      P(Node2) = max(P(有意),P(意|有)
      P(Node3) = max(P(Node1)\*P(意见|有),P(Node2)\*P(见|意))
      P(Node5) = P(Node3)\*P(分词|意见)

## 二元词典

二元词典的数据格式大致如下：

    一定@千倍 2

在Word结构体重，新增一个BiEntry结构体，这个结构体中保存了以当前词为前驱词的词语的出现次数。

```java
public class BiEntry {
    private Integer id;
    private Map<Integer, Integer> biMap;

    public BiEntry(Integer id) {
        this.id = id;
        biMap = new TreeMap<Integer, Integer>();
    }

    public Integer getId() {
        return id;
    }

    public void setId(Integer id) {
        this.id = id;
    }

    public Map<Integer, Integer> getBiMap() {
        return biMap;
    }

    public void setBiMap(Map<Integer, Integer> biMap) {
        this.biMap = biMap;
    }

    public void put(Integer id, Integer freq) {
        if (biMap != null) {
            biMap.put(id, freq);
        }
    }

    public Integer get(Integer id) {
        if (biMap != null) {
            Integer res = biMap.get(id);
            return res == null ? 0 : res;
        }
        return 0;
    }
}

```

二元词典代码如下，在加载完一元词典后，扫描二元词典：


```java
public class BiGramDictionary implements IBiGramDictionary, IDictionary {

    private IDictionary unaryDictionary;

    private int totalFreq = 0;

    public BiGramDictionary(String fileName, IDictionary dictionary) {
        this.unaryDictionary = dictionary;
        try {
            FileInputStream file;
            file = new FileInputStream(fileName);
            BufferedReader in = new BufferedReader(new InputStreamReader(file, "UTF-8"));
            String line;
            try {
                while ((line = in.readLine()) != null) {
                    //找到前后词的入口，并将联合出现的次数维护在前面词的表中
                    StringTokenizer st = new StringTokenizer(line, " ");
                    String words = st.nextToken();
                    String pre = words.substring(0, words.indexOf("@"));
                    String suf = words.substring(words.indexOf("@") + 1);
                    TSTNode preNode = unaryDictionary.getNode(pre);
                    if (preNode == null || preNode.getData() == null) continue;
                    TSTNode sufNode = unaryDictionary.getNode(suf);
                    if (sufNode == null || sufNode.getData() == null) continue;
                    Integer freq = Integer.valueOf(st.nextToken());
                    Integer id = preNode.getData().biEntry.getId();
                    sufNode.getData().biEntry.put(id, freq);
                    totalFreq += freq;
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
            in.close();
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (UnsupportedEncodingException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public Integer getBiFreq(Word prev, Word suf) {
        if (prev.biEntry == null || suf.biEntry == null) return 0;
        return prev.biEntry.get(suf.biEntry.getId());
    }

    public Word matchWord(String sentence, int offset) {
        return unaryDictionary.matchWord(sentence, offset);
    }

    public void matchAll(String sentence, int offset, List<Word> ret) {
        unaryDictionary.matchAll(sentence, offset, ret);
    }

    public Long getN() {
        return unaryDictionary.getN();
    }

    public TSTNode getNode(String a) {
        return unaryDictionary.getNode(a);
    }

    public int getTotalFreq() {
        return totalFreq;
    }
}

```

上述分词算法的具体分词算法实现如下：

```java
public class BiGramSegment implements ISegment {
    private BiGramDictionary biGramDictionary;

    private static final Word START = new Word("START", WordType.English, 1000);

    private static final Double MIN_PROB = Double.NEGATIVE_INFINITY;

    private static final Double LAMBDA_1 = 0.3;

    private static final Double LAMBDA_2 = 0.7;

    public BiGramSegment(BiGramDictionary biGramDictionary) {
        this.biGramDictionary = biGramDictionary;
    }

    public List<Word> seg(String sentence) {
        int len = sentence.length() + 1;
        double[] prob = new double[len];
        Word[] preWords = new Word[len];
        int[] preNode = new int[len];
        preWords[0] = START;
        prob[0] = 0.0;
        List<Word> ret = new ArrayList<Word>();
        List<Word> wordMatch = new ArrayList<Word>();

        for (int i = 1; i < len; i++) {
            Double maxProb = MIN_PROB;
            int maxPre = -1;
            Word preWord = null;
            //找到当前节点的所有前驱词
            biGramDictionary.matchAll(sentence, i - 1, wordMatch);
            for (Word w1 : wordMatch) {
                //遍历所有前驱词，计算前驱词的开始节点的最佳前驱词与当前前驱词的联合概率，并找到最大概率的前驱词作为当前节点的前驱词
                int start = i - w1.term.length();
                Word w2 = preWords[start];
                double wordProb;

                int biGramFreq = biGramDictionary.getBiFreq(w2, w1);
                //平滑之后的概率
                wordProb = LAMBDA_1 * w1.freq / biGramDictionary.getN() + LAMBDA_2 * (biGramFreq / w2.freq);

                double nodeProb = prob[start] + Math.log(wordProb);
                if (nodeProb >= maxProb) {
                    maxPre = start;
                    maxProb = nodeProb;
                    preWord = w1;
                }
            }
            prob[i] = maxProb;
            preWords[i] = preWord;
            preNode[i] = maxPre;
        }
        Stack<Integer> path = new Stack<Integer>();
        for (int i = sentence.length(); i > 0; ) {
            path.add(i);
            i = preNode[i];
        }
        while (!path.empty()) {
            ret.add(preWords[path.pop()]);
        }
        return ret;
    }
}
```
