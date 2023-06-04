---
layout: post
title: "TIL: Evaluator from Huggingface"
description: "This article describe the usage of evaluator from Huggingface"
---

TIL that Hugging Face has a library called `evaluate` which includes a useful
`Evaluator` class for quickly evaluating transformer models on datasets. I will
cover how to use this Evaluator class and also address a minor issue we
currently have, along with its workaround.

The following code is to load the `banking77` dataset and one of the models
that I fine-tuned and pushed to HF mode hub:

{% highlight python %}
!pip install -U transformers datasets evaluate

import evaluate
from evaluate import evaluator
from datasets import load_dataset
from transformers import AutoTokenizer, AutoModelForSequenceClassification

task_evaluator = evaluator("text-classification")
data = load_dataset("banking77")

tokenizer = AutoTokenizer.from_pretrained("lxyuan/banking-intent-distilbert-classifier")
model = AutoModelForSequenceClassification.from_pretrained("lxyuan/banking-intent-distilbert-classifier")

# currently this feature development is on-hold so we will use sklearn to do the evaluation
# https://discuss.huggingface.co/t/combining-metrics-for-multiclass-predictions-evaluations/21792/11
results = task_evaluator.compute(
    model_or_pipeline=model,
    data=data["test"],
    metric=evaluate.combine([
        "accuracy",
#        evaluate.load("precision", average="macro"),
#        evaluate.load("recall", average="macro"),
#        evaluate.load("f1", average="macro")
    ]),
    tokenizer=tokenizer,
    label_mapping=model.config.label2id,
    strategy="simple",
)

>>> {'accuracy': 0.9243506493506494,
 'total_time_in_seconds': 30.070161925999855,
 'samples_per_second': 102.42711720607363,
 'latency_in_seconds': 0.009763039586363589}

{% endhighlight %}

As mentioned in the code snippet that the development of `Evaluator` feature is
currently on-hold so it doesn't support precision/recall/f1 metrics for
multiclass problem. If you uncomment them and run the code, you will see error
message like:
{% highlight python %}
ValueError: Target is multiclass but average='binary'. Please choose another average setting, one of [None, 'micro', 'macro', 'weighted'].
{% endhighlight %}

The workaround is simply switch to sklearn classification report function, as follows:

{% highlight python %}
from transformers import pipeline
from sklearn.metrics import classification_report

text_classifier = pipeline("text-classification", model=model, tokenizer=tokenizer, device=0)

x_test = [text for text in data["test"]["text"]]
y_test = [label for label in data["test"]["label"]]
y_pred = text_classifier(x_test)
y_pred = [model.config.label2id[pred['label']] for pred in y_pred]

label_names = [label for id, label in model.config.id2label.items()]
report = classification_report(y_test, y_pred, target_names=label_names, digits=4)
print("Classification Report:\n", report)

>>>
Classification Report:
                                                   precision    recall  f1-score   support

                                activate_my_card     1.0000    0.9750    0.9873        40
                                       age_limit     0.9756    1.0000    0.9877        40
                         apple_pay_or_google_pay     1.0000    1.0000    1.0000        40
                                     atm_support     0.9750    0.9750    0.9750        40
                                automatic_top_up     1.0000    0.9000    0.9474        40
                                ...
                                   verify_top_up     1.0000    1.0000    1.0000        40
                        virtual_card_not_working     1.0000    0.9250    0.9610        40
                              visa_or_mastercard     0.9737    0.9250    0.9487        40
                             why_verify_identity     0.9118    0.7750    0.8378        40
                   wrong_amount_of_cash_received     1.0000    0.8750    0.9333        40
         wrong_exchange_rate_for_cash_withdrawal     0.9730    0.9000    0.9351        40

                                        accuracy                         0.9244      3080
                                       macro avg     0.9282    0.9244    0.9243      3080
                                    weighted avg     0.9282    0.9244    0.9243      3080
{% endhighlight %}

That's it!

Related:
- [Combining metrics for multiclass predictions evaluations](https://discuss.huggingface.co/t/combining-metrics-for-multiclass-predictions-evaluations/21792)
- [Using the evaluator](https://huggingface.co/docs/evaluate/base_evaluator)
- [lxyuan/banking-intent-distilbert-classifier](https://huggingface.co/lxyuan/banking-intent-distilbert-classifier)
