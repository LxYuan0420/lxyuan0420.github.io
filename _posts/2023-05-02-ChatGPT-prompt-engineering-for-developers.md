---
layout: post
title: "ChatGPT Prompt Engineering for Developers"
description: "This article describe some key takeaways concept that I learnt from the Coursera course."
---

This goal of this article is to summarise what I learnt from the Coursera
course: [ChatGPT Prompt Engineering for
Developers](https://www.deeplearning.ai/short-courses/chatgpt-prompt-engineering-for-developers/)
as well to share few prompt engineering articles that I enjoy reading. Hope
this is helpful for people who want to learn how to write good ChatGPT prompts
and have some hands-on prompt engineering experience.

From the course, we learn that ChatGPT is quite powerful and capable of
performing a lot of NLP tasks such as inferring, summarisation, transforming
(e.g., converting extracted information to JSON format) and others tasks. In
order to achieve good performance and getting desired output format, it is
important to follow these prompting principles: (1) Write clear and clear
instructions (short != clear) and (2) Give the model time to "think" and (3)
Use delimiters to clearly indicate distinct parts of the input. For the
principle (2), what it means is actually to ask ChatGPT to think or explain the
solution step by step.

In the following section, we will see examples that demostrate different kind
NLP tasks that ChatGPT is capable of and also to demostrate how to write good
prompts based on the principles that we described above.


{% highlight python %}

text = f"""
In a charming village, siblings Jack and Jill set out on \ 
a quest to fetch water from a hilltop \ 
well. As they climbed, singing joyfully, misfortune \ 
struck—Jack tripped on a stone and tumbled \ 
down the hill, with Jill following suit. \ 
Though slightly battered, the pair returned home to \ 
comforting embraces. Despite the mishap, \ 
their adventurous spirits remained undimmed, and they \ 
continued exploring with delight.
"""

question = f"""
Did Jack and Jill give up after the misfortune or did \
they continue with their adventure?
"""

prompt_1 = f"""
Perform the following actions:
1 - Summarize the following text delimited by triple \
backticks with 1 sentence.
2 - Translate the summary into French.
3 - List each name in the French summary.
4 - Output a json object that contains the following \
keys: french_summary, num_names.
5 - Answer question delimited by triple quotes

Separate your answers with line breaks.

Text:
```{text}```

Question:
\"\"\"{question}\"\"\"
"""
response = get_completion(prompt_1)
print("Completion for prompt 1:")
print(response)

{% endhighlight %}

ChatGPT answered: 

{% highlight python %}

Completion for prompt 1:
1 - Siblings Jack and Jill go on a quest to fetch water from a hilltop well,
but after Jack trips and tumbles down the hill with Jill following, they return
home slightly battered but with their adventurous spirits undimmed.

2 - Les frères et sœurs Jack et Jill partent en quête d'eau d'un puits au
sommet d'une colline, mais après que Jack a trébuché et est tombé en bas de la
colline avec Jill qui suit, ils rentrent chez eux légèrement meurtris mais avec
leurs esprits aventureux intacts.

3 - Jack, Jill

4 - {"french_summary": "Les frères et sœurs Jack et Jill partent en quête d'eau
d'un puits au sommet d'une colline, mais après que Jack a trébuché et est tombé
en bas de la colline avec Jill qui suit, ils rentrent chez eux légèrement
meurtris mais avec leurs esprits aventureux intacts.",
     "num_names": 2
   }

5 - They continued exploring with delight.

{% endhighlight %}

It was pretty impressive! Look at the example, we use delimiter to specify text
to be summarised and question to be answered. We write clear instruction and
describe the desired output format to ChatGPT. It performed 5 different tasks
accurately: Summarisation, translation, named entity extraction, transforming
and Question-answering.

Let's look at another example that demo the principle-(2): Think step by step. 

{% highlight python %}
prompt = """
I went to the market and bought 10 apples. 
I gave 2 apples to the neighbor and 2 to the repairman. 
I then went and bought 5 more apples and ate 1. 
How many apples did I remain with?

Let's think step by step."""

response = get_completion(prompt)
print(response)
{% endhighlight %}

{% highlight python %}
1. Bought 10 apples.
2. Gave 2 apples to the neighbor, remaining with 8 apples.
3. Gave 2 more apples to the repairman, remaining with 6 apples.
4. Bought 5 more apples, now having 11 apples.
5. Ate 1 apple, remaining with 10 apples.

Therefore, you remained with 10 apples.
{% endhighlight %}

Again, the answer and step-by-step explanation are correct. :thumbsup:

Personally, I'm interested in using ChatGPT to perform information extraction
on unstructure text data. Specifically, extracting key financial indicator
values such as Earning-per-share (EPS), dividend, net-asset-value (NAV) from
text. Let's try them:

{% highlight python %}
text = """
Subject to the approval by the shareholders at the forthcoming Annual General Meeting, the directors have proposed an exempt
(one-tier) final dividend of 3.10 cents per share (2020: exempt (one-tier) final dividend of 3.00 cents per share) amounting to an
estimated amount of $46.6 million (2020: $45.1 million) in respect of the year ended 31 December 2021. This proposed dividend
has not been included as a liability in the financial statements. The total dividends paid and proposed for the year, comprising the
interim and final dividend in respect of the year ended 31 December 2021 approximates 70.2% (2020: 70.5%) of the Group’s net
profit attributable to owners of the Company.
"""

prompt = f"""
Your task is to extract the following financial indicators \
from the text delimited by backticks. In this case, we only \
want financial indicators value from year 2021 and 2020.

If the financial indicator information is available, extract them \
and present them in a dataframe format with the following columns:

<type of indicator> <year> <value>

If the financial indicator information is not available, just answer \
information not found.

Financial indicator:
- Basic earning per share (cent)
- Diluted earning per share (cent)
- Dividend
- Number of share (NOS)
- Net asset value (NAV)

text: {text}
"""

response = get_completion(prompt)
print(response)

{% endhighlight %}

ChatGPT answered: 

--- 
<table>
  <tr>
    <th>Type of Indicator</th>
    <th>Year</th>
    <th>Value</th>
  </tr>
  <tr>
    <td>Diluted earning per share (cent)</td>
    <td>2021</td>
    <td>Information not found</td>
  </tr>
  <tr>
    <td>Basic earning per share (cent)</td>
    <td>2021</td>
    <td>Information not found</td>
  </tr>
  <tr>
    <td>Dividend</td>
    <td>2021</td>
    <td>3.10 cents per share</td>
  </tr>
  <tr>
    <td>Number of share (NOS)</td>
    <td>2021</td>
    <td>Information not found</td>
  </tr>
  <tr>
    <td>Net asset value (NAV)</td>
    <td>2021</td>
    <td>Information not found</td>
  </tr>
  <tr>
    <td>Diluted earning per share (cent)</td>
    <td>2020</td>
    <td>Information not found</td>
  </tr>
  <tr>
    <td>Basic earning per share (cent)</td>
    <td>2020</td>
    <td>Information not found</td>
  </tr>
  <tr>
    <td>Dividend</td>
    <td>2020</td>
    <td>3.00 cents per share</td>
  </tr>
  <tr>
    <td>Number of share (NOS)</td>
    <td>2020</td>
    <td>Information not found</td>
  </tr>
  <tr>
    <td>Net asset value (NAV)</td>
    <td>2020</td>
    <td>Information not found</td>
  </tr>
</table>
--- 

The results are all correct! The provided text is all about dividend and no
additional information for other indicators.

That's it. I recommend everyone to try the free prompt engineering course on
Coursera to get some hands-on experience and play with the notebooks. You don't
even have to generate your own openai api key and setting up the env. If you
wish to learn more about prompt engineering, please check out the following
articles or resources:
- [dair-ai/Prompt-Engineering-Guide](https://github.com/dair-ai/Prompt-Engineering-Guide)
- [Prompt Engineering vs. Blind Prompting](https://mitchellh.com/writing/prompt-engineering-vs-blind-prompting#user-content-fnref-4)
- [Prompt Engineering Guide](https://www.promptingguide.ai/introduction)
- [Prompt Engineering](https://lilianweng.github.io/posts/2023-03-15-prompt-engineering/)


