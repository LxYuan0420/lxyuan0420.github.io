---
layout: post
title: "DocQA: Chat with PDFs using LangChain and OpenAI"
description: "This article demostrate the query result of DocQA using LangChain and OpenAI"
---

There's been a lot of buzz lately about LLM and how it can be used for
different applications with OpenAI. In this article, we'll give you a quick
rundown on how to set up a cool DocQA application using LangChain and OpenAI,
and we'll even show you some examples of the query results. So, let's dive in!

The high-level idea is to:

Load PDFs from a directory.
Read all the texts and split them into chunks.
Convert each chunk into an embedding.
Save the embedding into ChromaDB.
Convert the user's query input into an embedding.
Find the most relevant chunk based on embedding similarity.
Output the selected chunk.

The code is pretty simple:

{% highlight python %}
import os
from langchain.document_loaders import PyPDFDirectoryLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.llms import OpenAI
from langchain.chains import ConversationalRetrievalChain
from langchain.vectorstores import Chroma
from langchain.embeddings import OpenAIEmbeddings
from langchain.chat_models import ChatOpenAI
from langchain.chains import RetrievalQA

loader = PyPDFDirectoryLoader("/content/my_pdf_dir")
docs = loader.load()

text_splitter = RecursiveCharacterTextSplitter(chunk_size=512, chunk_overlap=100)
texts = text_splitter.split_documents(docs)

persist_directory = "./my_pdf_db"
embeddings = OpenAIEmbeddings()
vectordb = Chroma.from_documents(documents=texts,
                                 embedding=embeddings,
                                 persist_directory=persist_directory)
vectordb.persist()

retriever = vectordb.as_retriever()
llm = ChatOpenAI(model_name='gpt-3.5-turbo') #gpt-4 or gpt-3.5-turbo
qa = RetrievalQA.from_chain_type(llm=llm, chain_type="stuff", retriever=retriever)

while True:
        user_input = input("Enter a query: ")
        if user_input == "exit":
            break

        query = f"###Prompt {user_input}"
        try:
            llm_response = qa(query)
            print(llm_response["result"])
            print("\n")
        except Exception as err:
            print('Exception occurred. Please try again', str(err))

{% endhighlight %}

Below is the output:
{% highlight python %}
>>>
Enter a query: Refer to Dragon Group International financial report, what is the reported "Cash and cash equivalent" amount as of 30 September 2022?
I'm sorry, I cannot find the reported "Cash and cash equivalent" amount as of 30 September 2022 in the given context. The condensed consolidated balance sheet available on page 20 provides some financial information, but it does not include this specific figure.


Enter a query: Refer to the Dragon Group financial report, where is the company located?
According to the financial report, Dragon Group International Limited is domiciled and incorporated in Singapore.


Enter a query: Has the company HS Optimus Holdings Limited proposed any interim dividend for the periods ending on September 30, 2022, and September 30, 2021, based on the given context
According to the given context, there is no proposal or declaration of any dividend by HS Optimus Holdings Limited for the six months ended 30 September 2022 or the corresponding period of the immediately preceding financial year.


Enter a query: Refer to the HS Optimus Holding Limited report, what is the reported net asset value per share as of 30 September 2022?
The reported net asset value per share based on existing issued share capital as of 30 September 2022 is 1.09 cents.


Enter a query: Refer to the HS Optimus Holding Limited report, what is the reported net asset value per share as of 31 Mar 2022?
The reported net asset value per share as of 31 Mar 2022 for HS Optimus Holding Limited is 1.16 cents.


Enter a query: What was the net book value of the assets disposed of by the Group during the six months ended September 30, 2022?
The net book value of the assets disposed of by the Group during the six months ended September 30, 2022 was $190.

{% endhighlight %}

###### Test-1: Extract single and large table.
{% include image.html path="documentation/camelot_textract_single_comparison.png" path-detail="documentation/camelot_textract_single_comparison.png" alt="Camelot vs AWS on single-table test" %}

Refer to the image above, left-hand-side is the Camelot output saved as csv file; middle is the AWS textract output; right-hand-side is the input pdf page. 

In this case, i think AWS produced better output for me: 
- (1) Capture the entire table;
- (2) Long line item are captured and grouped correctly (i.e., see the third underlined section title - Other comphrensions Income); 
- (3) Negative numbers are parsed as the original format (i.e., output as (9513) instead of -9513); 
- (4) Column header texts are grouped together (i.e., Mar 2022 $'000').

The third point is not necessarily an issue but personally i prefer original
format.

----

###### Test-2: Extract multi-tables.
{% include image.html path="documentation/camelot_textract_comparison.png" path-detail="documentation/camelot_textract_comparison.png" alt="Camelot vs AWS on multi-table test" %}
Similarly, camelot output is on the left hand side whereas AWS textract output
is the middle one. 

In this test case, AWS produced better result again: 
- (1) Long line item are grouped as one cell (look at last row of the first table.); 
- (2) Captured the second table at the bottom of the page. However, one noticable mistake from AWS
is that the `Note` column in the second table is merged into the left-hand-side
row-header column.

----


###### Image below is the screenshot of AWS textract UI.

{% include image.html path="documentation/aws-textract-ui.png" path-detail="documentation/aws-textract-ui.png" alt="AWS textract UI" %}

----

###### Sample Camelot code
{% highlight python %}
# installation 
!pip install "camelot-py[base]"

# fix: PyPDF2.errors.DeprecationError
# source: https://github.com/camelot-dev/camelot/issues/339
!pip install 'PyPDF2<3.0'

# fix: RuntimeError: Please make sure that Ghostscript is installed
# source: https://github.com/atlanhq/camelot/issues/184
!apt install ghostscript python3-tk
from ctypes.util import find_library
find_library("gs")

# fix: DependencyError: PyCryptodome is required for AES algorithm
!pip install pycryptodome==3.15.0

import camelot

# Test-1: single-table
table = camelot.read_pdf(
    "/content/SG220527OTHRMX7S-travelite_holdings_ltd.pdf", 
    pages='1', 
    flavor='stream'
)
# save as csv file
table[0].to_csv("single.csv")


# Test-2: multi-tables
table = camelot.read_pdf(
    "/content/SG220805OTHR7ZPG_Sembcorp Industries Ltd_20220805072353_00_FS_2Q_20220630.pdf",
    pages='6',
    flavor='stream'
)
# save as csv file
table[0].to_csv("multi.csv")
{% endhighlight %}


Related:
- [Discussion on HN](https://news.ycombinator.com/item?id=20470439)
- [OCR spelling correction is hard](https://maxhalford.github.io/blog/ocr-spelling-correction-is-hard/)
