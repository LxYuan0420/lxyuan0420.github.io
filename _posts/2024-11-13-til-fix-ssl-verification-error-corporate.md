---
layout: post
title: "TIL: How to Handle SSL Verification Errors in Corporate Environments"
description: "Handling SSL Verification Errors with Custom Certificates in Corporate Environments - ZScaler"
---

Today, I ran into a common issue in corporate networks: SSL verification errors when trying to use certain tools. In many corporate environments, network security solutions (like Zscaler, Palo Alto, etc.) intercept HTTPS connections for inspection. This can lead to SSL verification errors when working with tools like Python’s requests, curl, and the Hugging Face CLI, which expect standard SSL certificates.


If you have seen errors such as `Caused by SSLError(SSLError(1, '[SSL] record layer failure (_ssl.c:1020)]` or `Caused by SSLError(SSLError(1, '[SSL: WRONG_VERSION_NUMBER] wrong version number (_ssl.c:1000)'))`, this guide provides a simple solution to enable these tools to function with custom SSL certificates.


##### Why Does This Happen?

Corporate security solutions often insert their own certificates to inspect encrypted traffic. As a result, when tools like `requests` or `curl` try to verify SSL certificates, they encounter an unfamiliar certificate that isn’t in their trusted CA bundle, resulting in SSL errors.

To solve this, you’ll need to add the network’s root certificate to the CA bundle used by these tools.

##### Step-by-Step Guide to Setting Up Custom Certificates

Here’s how to configure custom certificates so that `requests`, `curl`, and other tools can connect to secure sites without issues. This approach ensures your tools recognize the certificate as trusted and prevents SSL verification errors.

First, obtain the custom root certificate from your IT department. It’s often a `.pem` file, typically named something like CompanyRootCertificate.pem. This certificate is needed to add to your system’s CA bundle.

{% highlight bash %}
# 1. Download Zscaler Root Certificate
# Make sure you have `ZscalerRootCertificate-2048-SHA256.pem` downloaded

# 2. Install `certifi` with Homebrew
brew install certifi

# 3. Add Zscaler Certificate to `certifi` Bundle
# Find the path to certifi's CA bundle and append Zscaler's certificate
cat ZscalerRootCertificate-2048-SHA256.pem >> $(python3 -m certifi)

# 4. Set Environment Variable for Python's `requests`
# Add this line to your ~/.zshrc or ~/.bashrc
echo 'export REQUESTS_CA_BUNDLE=$(python3 -m certifi)' >> ~/.zshrc
# Reload shell configuration
source ~/.zshrc  # or `source ~/.bashrc` for Bash users

# 5. Test Configuration

# Python's requests
python3 -c "import requests; print(requests.get('https://httpbin.org/get').status_code)"

# curl
curl https://httpbin.org/get

# Hugging Face CLI
huggingface-cli download google-bert/bert-base-uncased
{% endhighlight %}

Hopefully, this helps anyone else who runs into similar issues!
