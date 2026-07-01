During these day, 
whenever we explore about AI, RAG (retrieval augmented generation) are also becoming popular topic.

So we need to know why we need RAG and what is limit of the RAG if we cannot use correctly.

the first one, is we can have LLM in our software application, then why we need RAG ?

Because LLM are trained using generic knowledge , they can know general knowledge, but if you want very specific information like your own company data. 
or your specific industrial deep knowledge. Then LLM have limit. they cannot know that much.

so  you  want to let AI analyze also your company data, then how ?

So, you need to use RAG 

  1. you use one of the embedding llm model to convert your natural language data to numerical data 
     so that machine learning techniques can find the nearest neighbour.

---------------------------------------------------------------------------------------------------
---------------------------------------------------------------------------------------------------

Let's think you have laptop information, 
    eg. Dell laptop, HP Laptop, Apple Laptop

but your customer want to search notebook , 
    then in traditional software it cannot retrieve your laptop information.
    because customer type notebook, but your stored data is laptop.

This technique is known as keyword search.

    or Doctor or Physician
    or illness or disease
    or flight or aviation
    or car or automobile
    or apartment or flat

but you want to let your system auto know that notebook and laptop are semantically similar 
and let it retrieve laptop information to customer.

So, you need to use RAG to retrieve the correct information.

RAG can also help you to answer questions that are not in your data, but are related to your data.
     For example, if your customer asks "What is the best notebook for gaming?", 
     RAG can retrieve relevant information (laptop) from your data and generate an answer based on that.

So, the question is how RAG know it?

RAG always finding how much far or near your customer query data and your data.
Meaning RAG always calculate the distance in a high-dimensional vector space to get the nearest value.

---------------------------------------------------------------------------------------------------
---------------------------------------------------------------------------------------------------

Keyword search > Semantic Search > RAG > LLM;

loss functions > how wrong it is



