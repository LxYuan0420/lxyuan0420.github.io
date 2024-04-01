---
layout: post
title: "Deploying vLLMs with Docker: A Guide to Fixing NVIDIA GPU Access"
description: "A straightforward guide to resolving the 'unknown or invalid runtime name: nvidia' error for developers deploying VLLM using Docker."
---

vLLM, a high-throughput and memory-efficient inference engine, makes serving
large language models like LLaMA-2 straightforward, even on diverse platforms
like ROCm and various clouds via SkyPilot. This guide documents a Docker
deployment issue related to NVIDIA GPU access and provides a detailed solution,
enhancing the vLLM serving experience.

Deploying vLLMs with Docker service requires NVIDIA GPU support, but Docker can sometimes fail to access the GPU, showing errors. This guide resolves such issues, ensuring vLLM can utilize the necessary GPU resources.

### Problem

While deploying [vLLM](https://docs.vllm.ai/en/latest/serving/deploying_with_docker.html) with the following Docker command:

{% highlight bash %}
docker run --runtime nvidia --gpus all ...
{% endhighlight %}

I was met with:

{% highlight bash %}
docker: Error response from daemon: unknown or invalid runtime name: nvidia.
{% endhighlight %}

Removing `--runtime nvidia` led to a new error about the inability to find a
device driver with GPU capabilities.

### Resolution Steps:

#### Verifying nvidia-docker2 Installation

It's crucial to have `nvidia-docker2` installed for Docker to interface with
NVIDIA GPUs. Begin by verifying its presence:

{% highlight bash %}
# Check if nvidia-docker2 is installed
dpkg -l | grep nvidia-docker
docker info | grep nvidia
{% endhighlight %}

#### Installing nvidia-docker2

If nvidia-docker2 is missing, install it to bridge Docker with NVIDIA GPUs:
{% highlight bash %}
# Update package lists and install nvidia-docker2
sudo apt-get update
sudo apt-get install -y nvidia-docker2

# Restart Docker to apply changes
sudo systemctl restart docker
{% endhighlight %}

#### Confirming the Installation

Ensure the installation was successful by inspecting Docker's runtime
configuration and checking the installed version:

{% highlight bash %}
# Confirm Docker's runtime configuration for NVIDIA
cat /etc/docker/daemon.json

>>> {
    "runtimes": {
        "nvidia": {
            "path": "nvidia-container-runtime",
            "runtimeArgs": []
        }
    }
}

# Verify the nvidia-docker2 version
dpkg -l | grep nvidia-docker

# Check Docker runtimes for NVIDIA support
docker info | grep nvidia
{% endhighlight %}

### Successful vLLM Deployment

With NVIDIA GPU support enabled, execute the Docker command to deploy the vLLM:

{% highlight bash %}
# Deploy vLLM with NVIDIA GPU support
docker run --runtime nvidia --gpus all \
    -v /home/likxun/.cache/huggingface:/root/.cache/huggingface \
    --env "HUGGING_FACE_HUB_TOKEN=$(cat /home/likxun/.cache/huggingface/token)" \
    -p 8000:8000 \
    --ipc=host \
    vllm/vllm-openai:latest \
    --model TheBloke/Mistral-7B-Instruct-v0.1-GPTQ \
    --quantization "gptq" \
    --dtype "half"

{% endhighlight %}


#### Testing the Deployment

Verify that the vLLM is operational by executing a test request:

{% highlight bash %}
# Test the vLLM deployment
curl http://localhost:8000/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{
        "model": "TheBloke/Mistral-7B-Instruct-v0.1-GPTQ",
        "messages": [{"role": "user", "content": "What is 2+2?"}]
    }'
{% endhighlight %}

#### Expected Output:
{% highlight bash %}
{
    "id": "cmpl-afbe2ffa3e0d4779ba28ff8afae5b6a9",
    "object": "chat.completion",
    "created": 1711946527,
    "model": "TheBloke/Mistral-7B-Instruct-v0.1-GPTQ",
    "choices": [
        {
            "index": 0,
            "message": {
                "role": "assistant",
                "content": " 2+2 is 4."
            },
            "logprobs": null,
            "finish_reason": "stop",
            "stop_reason": null
        }
    ],
    "usage": {
        "prompt_tokens": 16,
        "total_tokens": 25,
        "completion_tokens": 9
    }
}
{% endhighlight %}

This response indicates that the vLLM deployment is successful and capable of processing requests.

### Conclusion

This guide provided a detailed walkthrough for resolving Docker and NVIDIA GPU
integration issues, ensuring a successful vLLM deployment. By following these
steps, users can overcome common hurdles, enabling efficient and effective
model serving with GPU support.


