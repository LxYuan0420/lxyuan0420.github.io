---
layout: post
title: "How to train spaCy multiclass multilabel text classifier"
description: "This article describes the steps of training spaCy text classifiers."
---

This afternoon, I stumbled across a spacy tweet on my Twitter timeline and
realised that I haven't been using or training spaCy model for a long time. So
I think it might be a good idea to train a simple spaCy text classification
model to refresh my memory. This is definitely not a new or hot thing in the AI
world since the field is moving so fast and everyone is talking about large
language models (LLMs), ChatGPT, and other big models.

This post aims to give a quick introduction to spaCy text classification models
for others and also to serve as a learning note for future use.

In this article, the term "multiclass" refers to a given example belonging to
only one positive class out of K number of classes, whereas "multilabel" means
a given example can belong to zero or more labels at the same time. For
instance, we use multiclass to classify animal types like [dog, cat, fish]
(obviously you can be dog and cat at the same time), while a multilabel task
can be used for movie categorisation, tagging a movie as [romance, funny]
simultaneously.

We can break down this classification task into 5 parts:
- Load dataset
- Dataset Conversion
- Generate Config
- Train
- Evalute

##### 1. Load dataset

In this example, we are training a Chinese Legal text classification model. The
[CAIL](https://huggingface.co/datasets/coastalcph/fairlex) dataset is a Chinese
legal NLP dataset for judgment prediction and contains over 1m criminal cases.
The dataset provides labels for relevant article of criminal code prediction,
charge (type of crime) prediction, imprisonment term (period) prediction, and
monetary penalty prediction. The goal is to predict how severe was the
committed crime with respect to the imprisonment term. We approximate crime
severity by the length of imprisonment term, split in 6 classes (0, <=12,
<=36, <=60, <=120, >120 months).

{% highlight python %}
from collections import Counter
from datasets import load_dataset

dataset = load_dataset("coastalcph/fairlex", "cail")

# quick check on the value count too
c = Counter(dataset["train"]["label"])
>>> Counter({
    0: 13122, 
    1: 41200, 
    3: 4160, 
    2: 17579, 
    4: 2268, 
    5: 1671
})

# the label mapping is:
{
    0: "0",
    1: "<=12",
    2: "<=36",
    3: "<=60",
    4: "<=120",
    5: ">120",
}
{% endhighlight %}


##### 2. Dataset Conversion

{% highlight python %}
from spacy.tokens import DocBin
from tqdm import tqdm

def convert(data, output):
    db = DocBin()
    docs = [] 
    cats = []

    for document in tqdm(data):
        docs.append(document["text"])
        cats.append(document["label"])
    print(f"Loaded {len(docs)} text and {len(cats)} labels")

    print(f"Process docs thru spacy nlp pipeline")
    docs = nlp.pipe(docs, disable=["ner", "parser"]) 

    print(f"Add cats into each sample doc")
    for doc, target_label_idx in tqdm(zip(docs, cats), total=len(cats)):  
        for idx, label in enumerate(labels):
            doc.cats[label] = 1 if idx == target_label_idx else 0

        db.add(doc)  

    print(f"Save spacy data as {output}")
    db.to_disk(output)

convert(dataset["train"], "./train.spacy")
convert(dataset["validation"], "./dev.spacy")
convert(dataset["test"], "./test.spacy")
{% endhighlight %}

Note that, for each sample we have to specify 1/0 for all possible labels in
each `doc.cat` attribute. For instance, you are doing sentiment analysis and
you have 3 possible labels: [positive, neutral, negative]. A positive sentiment
article should be prepared like this:

{% highlight python %}

labels = ["positive", "neutral", "negative"]

target_label = "positive"
doc = nlp("I love this movie")

for label in labels:
    doc.cat[label] = 1 if label != target_label else 0

doc.cat
>>> {"positive": 1, "neutral":0, "negative": 0}

{% endhighlight %}

##### 3. Generate Config

We use spacy init command to generate a default config file to train a
multiclass ensemble text classifer focusing on accuracy.

{% highlight bash %}
python -m spacy init config chinese_textcat.cfg \
    --lang zh \
    --pipeline textcat \ # specify textcat_multilabel for multilabel
    --optimize accuracy \ # accuracy: use ensemble (cnn+bow); effiency: use bow
    --gpu
{% endhighlight %}

##### 4. Train
{% highlight bash %}
python -m spacy train chinese_textcat.cfg \
    --paths.train ./train.spacy \
    --paths.dev ./dev.spacy \
    --output chinse_legal_textcat # saved model name
{% endhighlight %}


##### 5. Evaluate
{% highlight bash %}
python -m spacy evaluate \
    ./chinse_legal_textcat/model-best/ \ # path to saved model
    ./test.spacy \ #path to test set
    --gpu-id 0
{% endhighlight %}

We can also load the saved model and predict on unseen sample.

{% highlight python %}
# random sample from train split
# label: 1 (<=12)
text = (
    "公诉机关指控，2017年3月25日11时许，被告人陈向明在本市东城区地铁5号线磁器口站至崇文门站车厢内，从被害人徐某的挎包内，盗窃钱包1个，"
    "内有人民币351.4元、身份证、医保卡、医疗卡、工商银行储蓄卡1张、建设银行储蓄卡1张、农业银行卡1张等，其中钱包经鉴定价值人民币30元。 "
    "被告人陈向明后被抓获到案，赃物已起获并发还。  上述事实，被告人陈向明在审理过程中无异议，并有到案经过，被害人徐某的陈述，"
    "证人张某、刘某、赵某的证言，辨认笔录，鉴定意见，扣押、发还清单，照片，视听资料，常住人口基本信息，被告人陈向明的供述及其前科劣迹材料等证据予以证实，"
    "足以认定。"
)
nlp = spacy.load("./chinse_legal_textcat/model-best")
doc = nlp(text)
print(doc.cats,  "-",  text)
>>> {
    '0': 0.017592856660485268, 
    '<=12': 0.9379693865776062, 
    '<=36': 0.018389828503131866, 
    '<=60': 0.005402231123298407, 
    '<=120': 0.016701236367225647, 
    '>120': 0.0039446111768484116
} 
- 公诉机关指控，2017年3月25日11时许，被告人陈向明在本市东城区地铁5号线磁器口站至崇文门站车厢内，
从被害人徐某的挎包内，盗窃钱包1个，内有人民币351.4元、身份证、医保卡、医疗卡、工商银行储蓄卡1张、
建设银行储蓄卡1张、农业银行卡1张等，其中钱包经鉴定价值人民币30元。
被告人陈向明后被抓获到案，赃物已起获并发还。
上述事实，被告人陈向明在审理过程中无异议，并有到案经过，被害人徐某的陈述，证人张某、刘某、赵某的证言，
辨认笔录，鉴定意见，扣押、发还清单，照片，视听资料，常住人口基本信息，被告人陈向明的供述及其前科劣迹材料等证据予以证实，
足以认定。

{% endhighlight %}

That's it. Check out the following notebooks for more information:
- [Toxic Comment Multilabel Classification using spaCy v3 TextCategorizer](https://github.com/LxYuan0420/nlp/blob/main/notebooks/Toxic_Comment_Multilabel_Classification_using_spaCy_v3_TextCategorizer.ipynb)
- [Chinese Legal Text Classification with spaCy](https://github.com/LxYuan0420/nlp/blob/main/notebooks/Chinese_Legal_NLP_Text_Classification_with_spaCy.ipynb)
