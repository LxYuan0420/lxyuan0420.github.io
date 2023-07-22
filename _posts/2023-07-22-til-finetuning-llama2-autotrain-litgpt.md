---
layout: post
title: "TIL: Finetuning Llama-2 Model with Autotrain and Lit-GPT"
description: "This article describe how to finetune the Llama-2 Model with two APIs"
---

Today, I'm going to share what I learned about fine-tuning the Llama-2 model
using two distinct APIs: autotrain-advanced from Hugging Face and Lit-GPT from
Lightning AI. This guide will be a blend of technical precision and
straightforward instructions, peppered with code examples to make the process
as clear as possible.


#### Fine-Tuning with autotrain-advanced from Hugging Face
Hugging Face's autotrain-advanced is a powerful tool that simplifies the
process of fine-tuning models. Here's a step-by-step guide on how to use it
with the Llama-2 model:

{% highlight python %}
# install the autotrain-advanced package and update PyTorch
!pip install -U autotrain-advanced --quiet
!autotrain setup --update-torch
# fine-tune the Llama-2 model
!autotrain llm --train \
    --project_name llama-2-7b-finetuned-alpaca \
    --data_path tatsu-lab/alpaca \
    --model abhishek/llama-2-7b-hf-small-shards \
    --learning_rate 2e-4 \
    --num_train_epochs 3 \
    --train_batch_size 4 \
    --eval_batch_size 4 \
    --gradient_accumulation_steps 32 \
    --use_peft \
    --use_int4 \
    --trainer sft \
    --push_to_hub \
    --save_total_limit 2 \
    --repo_id lxyuan/llama-2-7b-alpaca

{% endhighlight %}

This command will fine-tune the Llama-2 model on the Alpaca dataset from Tatsu
Lab, using a learning rate of 2e-4, a batch size of 4 for both training and
evaluation, and a gradient accumulation step of 32. The fine-tuned model will
be saved in the specified repository on Hugging Face's Model Hub.

The command above takes about 175 hours to complete training on a T4 GPU (free
colab use T4 too). I'm currently experiencing difficulties with my Google Cloud
Compute instance due to an error message that indicates the unavailability of
an n1-standard-8 VM instance in the asia-southeast1-b zone. Given this
situation, I plan to postpone my model training task to sometime next week. At
that time, I will be attempting the training on an A100 machine instead.


#### Fine-Tuning with Lit-GPT from Lightning AI

{% highlight python %}

# Download the model weights 
python scripts/download.py \ 
    --repo_id meta-llama/Llama-2-7b-hf 

# Convert the weights to Lit-GPT format 
python scripts/convert_hf_checkpoint.py \ 
    --checkpoint_dir checkpoints/meta-llama/Llama-2-7b-hf 

# Prepare the dataset 
# check out this script: https://github.com/Lightning-AI/lit-gpt/blob/main/scripts/prepare_alpaca.py
# to modify the Alpaca script, open `prepare_alpaca.py` and edit the prepare function. 
python scripts/prepare_alpaca.py \ 
    --destination_path data/dolly \ 
    --checkpoint_dir checkpoints/meta-llama/Llama-2-7b-hf 

# Finetune Llama 2 on custom dataset 
python finetune/lora.py \ 
    --data_dir data/dolly \ 
    --checkpoint_dir checkpoints/meta-llama/Llama-2-7b-hf 

{% endhighlight %}

For Step-3 of preparing the datset, we suggest reader to go thru this [blog](https://lightning.ai/blog/how-to-finetune-gpt-like-large-language-models-on-a-custom-dataset/) and this python [script](https://github.com/Lightning-AI/lit-gpt/blob/main/scripts/prepare_alpaca.py#L27) first to understand how to prepare your custom dataset.

That's it!

Related:
- [LLAMA-v2 training successfully on Google Colab's free version! "pip install autotrain-advanced"](https://twitter.com/abhi1thakur/status/1681641346396835841)
- [The EASIEST way to finetune LLAMA-v2 on local machine!](https://www.youtube.com/watch?v=3fsn19OI_C8)
- [How To Finetune GPT Like Large Language Models on a Custom Dataset](https://lightning.ai/blog/how-to-finetune-gpt-like-large-language-models-on-a-custom-dataset/)
- [Finetune Llama 2 on a custom dataset in 4 steps using Lit-GPT.](https://twitter.com/LightningAI/status/1682392395655118848)
