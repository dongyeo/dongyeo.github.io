---
layout:     post
title:      "徒手撸个NLP分词器（4）"
date:       2017-10-06 00:00:00 +0800
author:     "DongYeo"
header-img: "img/post-bg-02.jpg"
tags: ["NLP"]
---

 你吃过猪肉，但你会做么？———尼古拉斯·造轮子·董

（注：本系列文章为 [《自然语言处理原理与技术实现》](https://www.amazon.cn/%E5%9B%BE%E4%B9%A6/dp/B01G8JOUSO/ref=sr_1_5?ie=UTF8&qid=1493475144&sr=8-5&keywords=%E8%87%AA%E7%84%B6%E8%AF%AD%E8%A8%80%E5%A4%84%E7%90%86) 学习笔记，点击链接可购买，购买前请看评论哈~）

# 基于隐马尔科夫的词性标注

在完成分词以后，我们还需要标注出切分出来的各个词语的词性。我们可能会选出一个词语的词性最大可能性的词语，标注为该词语的词性，例如，改革这个词语，通常就会被标注为动词。

但是直接这么标注词语，显然不够准确，因为一个词语通常都有它的上下文，例如：推进改革，应该标注为名词，而改革技术会被标注为动词，这样的特征就叫做上下文特征。

词性标注有两种方法，一种是隐马尔科夫（Hidden Markov Model），一种是基于转换的学习方法，两个方法都考虑了频率和上下文两个要素。隐马尔科夫考虑了词性之间的转移概率和词性到词语的发射概率。


## 概率模型

假如给定词序列 W = w1w2w3...wi，目标词性序列 T= t1t2t3...ti，我们要使得 P(T\|W)的概率最大化。

使用贝叶斯公式，我们得出 P(T\|W) = P(T)\*P(W\|T)/P(W)，而P(W)可以忽略，于是，求目标概率的最大值就是求 P(T)\*P(W\|T)的最大值。

其中P(T) = P(t1t2t3...ti) = P(t1)P(t2\|t1)P(t3\|t1t2)...P(ti\|t1t2t3...ti-1),我们做独立性假设，使用2元模型做近似值，那么：

    P(T)≈P(t1)*P(t2|t1)....*P(ti|ti-1)

而P(W\|T)我们假设每一个词性生成词语的概率之间是独立的，即：

    P(W|T)≈P(w1|t1)*P(w2|t2)...P(wi|ti)

我们得出最终的概率计算公式是：

    P(T|W)≈P(t1)*P(t2|t1)....*P(ti|ti-1)*P(w1|t1)*P(w2|t2)...P(wi|ti)

对于隐马尔科夫模型，w是已知的，我们称之为显状态，词性是未知的，我们支持位隐状态，隐状态到显状态的概率我们称之为发射概率，隐状态之间的概率我们称之为转移概率，一个显状态可能有多种隐状态。那么上述的公式就是每个显状态可能出现的隐状态序列的发射概率和转移概率的乘积。例如：我爱你 就是计算以下几种组合的概率最大值：

    P(我|rr)*P(n|rr)*P(爱|n)*P(rr|n)*P(你|rr)
    P(我|rr)*P(v|rr)*P(爱|v)*P(rr|v)*P(你|rr)

如图，展示了如何寻找这个最大概率，其中每个箭头代表词性(隐状态)之间的转移，而每个节点代表该词性发射到对应词语(显状态)，我们遍历每个显状态，计算遍历每个显状态的隐状态并计算上述的概率，采用动态规划找到每个隐状态的最佳前驱隐状态，直至结束，最终通过回溯找到最佳的词性标注序列。

![bias]({{ $baseurl}}/img/vertbi.png)


## 转移矩阵词典

隐状态之间的转移概率我们可以通过通次语料库词性的前后关系计算得出，如下，我们称之为词性转移矩阵，值得注意的是，除了开始和结束词，每个词性的行总和与列总和应该相等，因为一个词被转移的数和转移出去的数应该相等，且等于该词性的出现次数，并且转移概率等于 matrix[from][to]/total[from]。

 词性|start|n|rr|V|end
---|---|----|---|---|
start|x|x|x|x|x
n|x|x|x|x|x
rr|x|x|x|x|x
v|x|x|x|x|x
end|x|x|x|x|x

转移矩阵词典代码实现：

```java
public class TransferMatrixDic {
    //nature ordinal 最大值
    private int ordinalMax;
    //每个词性（隐状态）的出现次数
    private int[] total;
    //转移矩阵
    private int[][] matrix;
    //所有词性出现的个数
    private int totalCount;
    //隐状态
    private int[] states;
    //隐状态起始概率
    private double[] start_probability;
    //隐状态转移概率矩阵
    private double[][] transition_probability;

    public TransferMatrixDic(String path) {
        try {
            BufferedReader br = new BufferedReader(new InputStreamReader(new FileInputStream(path), "UTF-8"));
            //读取第一行，读取表头，计算出最大的ordinal，生成隐状态数组
            String line = br.readLine();
            String[] header = line.split(",");
            String[] natures = new String[header.length - 1];
            System.arraycopy(header, 1, natures, 0, natures.length);
            int[] ordinalArray = new int[natures.length];
            ordinalMax = 0;
            for (int i = 0; i < ordinalArray.length; i++) {
                ordinalArray[i] = Enum.valueOf(Nature.class, natures[i]).ordinal();
                ordinalMax = Math.max(ordinalMax, ordinalArray[i]);
            }
            ordinalMax++;
            //生成矩阵
            matrix = new int[ordinalMax][ordinalMax];
            //初始化
            for (int i = 0; i < ordinalMax; i++) {
                Arrays.fill(matrix[i], 0);
            }
            //读取矩阵
            while ((line = br.readLine()) != null) {
                String[] params = line.split(",");
                int currentOrdinal = Enum.valueOf(Nature.class, params[0]).ordinal();
                for (int i = 1; i < ordinalArray.length; i++) {
                    matrix[currentOrdinal][ordinalArray[i]] = Integer.valueOf(params[i]);
                }
            }
            br.close();
            total = new int[ordinalMax];
            totalCount = 0;
            for (int i = 0; i < ordinalMax; i++) {
                total[i] = 0;
                for (int j = 0; j < ordinalMax; j++) {
                    total[i] += matrix[i][j];
                }
                if (total[i] == 0) {
                    //当前状态是一个结束词，没有当前状态转移出去的状态，所以要计算所有转移到当前状态下总和
                    //按照列累加
                    for (int z = 0; z < ordinalMax; z++) {
                        total[i] = matrix[z][i];
                    }
                }
                totalCount += total[i];
            }
            states = ordinalArray;
            start_probability = new double[ordinalMax];
            for (int state : states) {
                double count = total[state] + 1e-8; //平滑
                start_probability[state] = Math.log(count / totalCount);
            }
            transition_probability = new double[ordinalMax][ordinalMax];
            for (int from : states) {
                for (int to : states) {
                    double count = matrix[from][to] + 1e-8;
                    transition_probability[from][to] = Math.log(count / total[from]);
                }
            }
        } catch (UnsupportedEncodingException e) {
            e.printStackTrace();
        } catch (FileNotFoundException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public double getFrom(Nature from, Nature to) {
        return transition_probability[from.ordinal()][to.ordinal()];
    }

    public double getFrom(String from, String to) {
        return getFrom(Enum.valueOf(Nature.class, from), Enum.valueOf(Nature.class, to));
    }

    public int getNatureFreq(String nature) {
        return getNatureFreq(Enum.valueOf(Nature.class, nature));
    }

    public int getNatureFreq(Nature nature) {
        return total[nature.ordinal()];
    }

}

```

至于隐状态到显状态的发射概率，则是词语对应词性出现的次数/词性总次数，这在词典加载的时候就需要多记录词性的信息，对应的词典源文件的格式如下：

    爱 n 100 v 230

对应词语Word结构体重新增如下两个字段，用以记录词性对应出现的次数：

    public Nature[] natures;
    public int[] frequency;

最终，词性分词的代码实现如下：

```java
public class BasicTokenizer implements Tokenizer {

    private ISegment segment;
    private TransferMatrixDic transferMatrixDic;

    private Word START;

    private Word END;

    public BasicTokenizer(String unaryFileName, String binaryFileName, String matrixFileName) {
        TernaryNatureDic ternaryNatureDic = new TernaryNatureDic(unaryFileName);
        BiGramDictionary biGramDictionary = new BiGramDictionary(binaryFileName, ternaryNatureDic);
        this.transferMatrixDic = new TransferMatrixDic(matrixFileName);
        this.segment = new BiGramSegment(biGramDictionary);
        this.START = new Word(1) {
            {
                this.natures[0] = Nature.begin;
                this.frequency[0] = transferMatrixDic.getNatureFreq(Nature.begin);
            }
        };
        this.END = new Word(1) {
            {
                this.natures[0] = Nature.end;
                this.frequency[0] = transferMatrixDic.getNatureFreq(Nature.end);
            }
        };
    }

    public List<Term> segment(String sentence) {
        //先使用二元分词，得出显状态序列
        List<Word> words = new ArrayList<Word>();
        //添加虚拟开始和结束节点
        words.add(START);
        words.addAll(segment.seg(sentence));
        words.add(END);
        int ordinalMax = Nature.values().length;
        int stageSize = words.size();
        //累计概率
        double prob[][] = new double[stageSize][ordinalMax];
        //最佳前驱词性
        Nature bestPre[][] = new Nature[stageSize][ordinalMax];
        for (int i = 0; i < stageSize; i++) {
            Arrays.fill(prob[i], -1.0 / 0.0);
        }
        prob[0][Nature.begin.ordinal()] = 1;
        for (int stage = 1; stage < stageSize; stage++) {
            //第一层循环，遍历当前显状态的所有隐状态
            Word currentWord = words.get(stage);
            for (int i = 0; i < currentWord.natures.length; i++) {
                //遍历当前隐状态，计算出发射概率
                int currentFreq = currentWord.frequency[i];
                Nature currentNature = currentWord.natures[i];
                double emiProb = Math.log((double) currentFreq / transferMatrixDic.getNatureFreq(currentNature));
                //向前遍历，计算出所有的转移概率
                Word preWord = words.get(stage - 1);
                for (int j = 0; j < preWord.natures.length; j++) {
                    Nature preNature = preWord.natures[j];
                    double transProb = transferMatrixDic.getFrom(preNature, currentNature);
                    double preProb = prob[stage - 1][preNature.ordinal()];
                    //前隐状态概率+转移概率+发射概率
                    double currentProb = preProb + transProb + emiProb;
                    if (currentProb >= prob[stage][currentNature.ordinal()]) {
                        //更新当前隐状态概率和最贱前驱词性
                        prob[stage][currentNature.ordinal()] = currentProb;
                        bestPre[stage][currentNature.ordinal()] = preNature;
                    }
                }

            }
        }
        //回溯
        List<Term> terms = words.stream().map(word -> {
            Term term = new Term();
            term.setTerm(word.term);
            return term;
        }).collect(Collectors.toList());
        Nature natureTemp = Nature.end;
        for (int i = stageSize - 1; i > 1; i--) {
            natureTemp = bestPre[i][natureTemp.ordinal()];
            terms.get(i - 1).setNature(natureTemp);
        }
        return terms.stream().skip(1).limit(stageSize - 2).collect(Collectors.toList());
    }
}

```
