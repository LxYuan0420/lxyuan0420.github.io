---
layout: post
title: "Fix: Failure to Start Jupyter Lab Notebook"
description: "This article describes how to fix the issue of being unable to start JupyterLab"
---

Yesterday, I encountered a minor issue while attempting to launch a Jupyter Lab notebook on a new cloud setup. The notebook wouldn't initiate because it was unable to locate a component called 'nbclassic'. Below, you'll find the terminal output and the solution to this problem:

###### Terminal output
{% highlight bash %}
(env) root@lx:~/playgrounds# jupyter Lab --no-browser
[1] 65423
(env) root@lx:~/playgrounds# [I 2023-87-29 12:86:31.035 ServerApp] Package jupyterlab took 0.00085
[I 2023-87-29 12:06:31.043 ServerApp] Package jupyter_Isp took 8.0873s to import
[W 2023-07-29 12:06:31.043 -jupyter-server-extension-points' function was not found in jupyter_Isp.
nction name well de aeprecated in future releases of
Jupyter Server.
[I 2023-87-29 12:06:31.046 ServerApp] Package jupyter_server_terminals took 0.0032s to import
ГІ 2023-07-29 12:06:31.846 ServerAppi |
Package notebook_shim took 0.0228s to import
W 2023-07-29 12:86:31.846 ServerappIA jupyter server extenston points function was not found in notel function name
WeLL de deprecacca in Tucure receases o
• Jupycer server.
[I 2023-87-29 12:06:31.047 ServerApp] jupyter-_Isp | extension was successfully linked.
2023-07-29
Serverappa jupyter_server_terminals extension was successfully linked
1 2023-01-29 16:06:31.034 Serverappjupyterlab I
2023-07-29 12:06:31.210 ServerApp] notebook_shim | extension was successfully linked.
[I 2023-07-29 12:06:31.232 ServerApp] notebook_shim | extension was successfully loaded.
[I 2023-87-29 12:86:31.234 ServerApp] jupyter_Isp | extension was successfully loaded.
[I 2023-07-29 12:06:31.235 ServerApp] jupyter_server_terminals | extension was successfully loaded.
[I 2023-07-29 12:06:31.236 LabApp] JupyterLab extension loaded from /root/playgrounds/env/lib/python3.8/site-packages/jupyterlab
[I 2023-07-29 12:06:31.236 LabApp]
application alrectory ls /root/ playgrounas/ env/ snare upyter/ Lat
[ 2023-07-29 12:06:31 236 LahAnni Extension Manager is 'pypi'
2023-01-29 12:06:31.239 Serverapp jupyterlab | extension was successtulLy Loaded.
Running as root is not recommendea. Use --allow-root to bypass]


{% endhighlight %}

###### Fix it by installing nbclass
{% highlight bash %}
(env) root@lx:~/playgrounds# pip freeze | grep nb
nbclient==0.8.0
nbconvert=7.7.3
nbformat==5.9.1
(env) root@lx:~/playgrounds# pip freeze | grep nbclassic 
(env) root@lx:~/playgrounds# pip3 install nbclassic

# launch it again
(env) root@lx:~/playgrounds# jupyter lab --allow-root --port 9000 --no-browser & 
{% endhighlight %}

Personal note: 
As a frequent user of JupyterLab and the JupyterLab-vim extension, I've
encountered numerous package incompatibility issues. These can be quite a
hassle, disrupting the smooth operation of the environment. The most
straightforward solution I've found is to pin and install specific versions of
these packages that are known to work well together. For instance, I've had
success with JupyterLab version 3.6.5 and JupyterLab-vim version 0.16.0. 
