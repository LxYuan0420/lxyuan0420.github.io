---
layout: post
title: "AWS Endpoint: Multi-model vs Multi-container"
description: "This article briefly explains the difference between multi-model and multi-container AWS endpoint deployment"
---

Amazon SageMaker is a managed machine learning service that provides a variety
of features for building, training, and deploying machine learning models. One
of these features is the ability to create endpoints, which are hosted
instances of machine learning models that can be used to make predictions.

When deploying a machine learning model to an endpoint, you have two options:
multi-model and multi-container.

Multi-model endpoints (MME) allow you to deploy multiple models in a single
container and share the same endpoint. This is useful if you have multiple
models that come from the same ML framework, the same algorithm, and perform
the same tasks (e.g., image classification). Referring to this
[example](https://sagemaker-examples.readthedocs.io/en/latest/advanced_functionality/multi_model_xgboost_home_value/xgboost_multi_model_endpoint_home_value.html#Create-the-Amazon-SageMaker-MultiDataModel-entity),
we just need to specify the S3 directory that contains all the models that
SageMaker multi-model endpoints will use to load and serve predictions. We can
see that the S3 prefix we specified when setting up MultiDataModel now has
multiple model artifacts. As such, the endpoint can now serve up inference
requests for these models. For instance:

{% highlight python %}
list(mme.list_models())
>>> ['Chicago_IL.tar.gz',
'Houston_TX.tar.gz',
'LosAngeles_CA.tar.gz',
'NewYork_NY.tar.gz'
]

# use target_model to specify model to call
predicted_value = predictor.predict(data=gen_random_house()[1:], target_model="Chicago_IL.tar.gz")

predicted_value = predictor.predict(data=gen_random_house()[1:], target_model="Houston_TX.tar.gz")

{% endhighlight %}

However, it is worth mentioning that it is possible to deploy models from
different framework backends to SageMaker MME if they can use the same
container image, as demonstrated in this
[example](https://github.com/aws/amazon-sagemaker-examples/blob/main/multi-model-endpoints/mme-on-gpu/cv/resnet50_mme_with_gpu.ipynb).
Notice that they create one container and set `Mode` to `MultiModel`, and
similarly use `TargetModel` to specify which model to use.

Multi-container endpoints allow you to deploy multiple models in separate
containers and share the same endpoint. This is useful if you have multiple
models that are different from each other, such as a model for image
classification and a model for natural language processing. Another common use
case of this is to deploy multiple HF models into different containers, and
each of them uses different environment variables. For instance:

{% highlight python %}
textClassificationModel = {
    'Image': hf_inference_dlc,
    'ContainerHostname': 'textClassificationModel',
    'Environment': {
	    'HF_MODEL_ID':'SamLowe/roberta-base-go_emotions',
	    'HF_TASK':'text-classification'
    }
}

zeroShotModel = {
    'Image': hf_inference_dlc,
    'ContainerHostname': 'zeroShotModel',
    'Environment': {
	    'HF_MODEL_ID':'facebook/bart-large-mnli',
	    'HF_TASK':'zero-shot-classification'
    }
}
{% endhighlight %}


Additional note on zipping model:

If you search online, you will find Python code that loads the model and
tokenizer from the Hugging Face model hub, and utilizes the tarfile Python
library. However, I would suggest handling the model cloning and zipping using
terminal commands to ensure the tar file is created correctly. Doing it with
Python code was nasty and wasted a few hours of my time.

{% highlight bash %}
$ git lfs install
$ git clone git@hf.co:lxyuan/distilgpt2-finetuned-finance #replace it with other models from hf hub
$ cd distilgpt2-finetuned-finance # make sure you cd into the dir
$ tar zcvf model.tar.gz *
$ aws s3 cp model.tar.gz <s3://{my-s3-path}>
{% endhighlight %}


Please read the docs mentioned below for better understanding:

Related:
- [Multi-model: Amazon SageMaker Multi-Model Endpoints using XGBoost](https://sagemaker-examples.readthedocs.io/en/latest/advanced_functionality/multi_model_xgboost_home_value/xgboost_multi_model_endpoint_home_value.html)
- [Multi-model: Run mulitple deep learning models on GPUs with Amazon SageMaker Multi-model endpoints (MME)](https://github.com/aws/amazon-sagemaker-examples/blob/main/multi-model-endpoints/mme-on-gpu/cv/resnet50_mme_with_gpu.ipynb)
- [Difference between Multi-model and multi-container](https://stackoverflow.com/questions/68913914/why-use-multi-container-endpoints-instead-of-multi-model-endpoints)
- [Multi-Container Endpoints with Hugging Face Transformers and Amazon SageMaker](https://www.philschmid.de/sagemaker-huggingface-multi-container-endpoint)
- [Available images](https://github.com/aws/deep-learning-containers/blob/master/available_images.md)
- [Hugginface DLC](https://huggingface.co/docs/sagemaker/reference#inference-dlc-overview)
