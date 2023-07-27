---
layout: post
title: "Running Llama2.c Inference: A Step-by-Step Guide"
description: "This article describe how to run inference on Llama2.c"
---

Today, we're diving into the world of llama2.c, a super cool project done by
Andrej Karpathy. We'll walk you through how to run inferencing using llama2.c.

#### The All-in-One Code Snippet
Here's a handy shell script that includes all the steps we're going to cover. Each line is preceded by a comment that explains what it does:

{% highlight bash %}
# Clone the llama2.c repository from GitHub
git clone https://github.com/karpathy/llama2.c.git

# Navigate into the llama2.c directory
cd llama2.c

# Download the model binary file and save it in the 'out' directory
wget https://karpathy.ai/llama2c/model.bin -P out

# Compile the run.c file with optimization level 3 and output the executable file named 'run'
gcc -O3 -o run run.c -lm

# Execute the 'run' file with 'model.bin' as an argument to start the model
./run out/model.bin
{% endhighlight %}

Sample output:

{% highlight bash %}
<s>
 Once upon a time, in a quiet town, there was a boy named Tim. Tim loved to play with his colorful toys. He had a big box filled with all the toys he would pick from the display.
 One day, Tim's mom told him to return the toys to the bin. Tim looked at the bin and thought about what he did. He knew it was not his toys, but he did it anyway. He put his toys in the bin and carried it home.
 At home, Tim's mom was happy to see the organized toys. She thanked Tim for returning the toys. Tim felt good that he did not take what his mom asked. From that day on, Tim became the responsible parent of the town.
 <s>
  Once upon a time, there was a little girl named Lily. One day, Lily was playing hide-and-seek with her friends. She was hiding behind a cabinet, but her friends couldn't find her. She was hidden so well that she couldn't come out of her hiding spot.
  Suddenly, she saw a door that was open. She went inside and found herself in a room with a big door. She tried to remember
  achieved tok/s: 53.178230
{% endhighlight %}

And that's all there is to it! You're now ready to run your very own llama2.c inference. Happy coding!

Related:
- [Official Llama2.c repo](https://github.com/karpathy/llama2.c)
- [Karpathy's Llama2.c - Quick Look for Beginners](https://www.youtube.com/watch?v=h6qZM76eOYE)
