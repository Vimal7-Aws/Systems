# **BERT in AI and LangGraph**

**BERT (Bidirectional Encoder Representations from Transformers)** is a Transformer-based language model developed by Google in 2018.

It is primarily designed to **understand the meaning and context of text**, rather than generate text like GPT, Gemini or Claude.

BERT is particularly useful for:

- Text classification.
- Semantic similarity.
- Intent classification.
- Named Entity Recognition (NER).
- Question answering.
- Information retrieval.
- Text embeddings (through BERT-based variants).

Let's understand it from an AI engineering and LangGraph perspective.

## **1. How does BERT work?**

BERT uses the Transformer architecture, specifically the **Encoder** part of the Transformer.

There are two major Transformer components:

**Transformer Encoder**

Understands and represents input text

BERT uses this

**Transformer Decoder**

Generates the next token based on previous tokens

GPT-style models use this

The important feature of BERT is bidirectional attention.

For example:

"The customer deposited money into the bank."

BERT understands the word `bank` using the surrounding words, including words appearing before and after it.

Compare this with traditional left-to-right language modelling, which processes context in one direction.

### **BERT architecture**

[SBERT vs BERT for Sentence Similarity: Comparison of the two models | by Eka Putra | Medium](https://images.openai.com/static-rsc-4/T5W9ZcJkhw_45i3YQwcrYrCKWINX-hWjQaW19maN6VF7xWwylkzj9xUpDReFRdqsI3RvjKYKdOj0q8y9m9TeZ5v8SEW92vnN0QGK2-7VD34KuenEMhPpH5hajt9tPfzp7bJ6LwgeGJcinV448AWX_UbNobu4yDvIpxq5fO_R3kQ?purpose=inline)

[medium.com](https://medium.com/%40eikhapoetra/sbert-vs-bert-for-sentence-similarity-comparison-of-the-two-models-933f5c097492?utm_source=chatgpt.com)

The main components are:

1. Token embeddings – convert tokens into numerical vectors.
2. Position embeddings – represent token positions.
3. Segment embeddings – distinguish sentence A and sentence B.
4. Multi-head self-attention – understand relationships between tokens.
5. Feed-forward layers – transform contextual representations.
6. Output representations – produce contextual embeddings for each token.

## **2. BERT vs GPT vs embedding models**

This distinction is important when designing production RAG systems.

| Feature             | BERT                              | GPT           | Embedding models        |
| ------------------- | --------------------------------- | ------------- | ----------------------- |
| Architecture        | Encoder                           | Decoder       | Usually encoder-based   |
| Main purpose        | Understand text                   | Generate text | Convert text to vectors |
| Text generation     | Not its primary purpose           | Yes           | No                      |
| Classification      | Excellent                         | Possible      | Not directly            |
| Semantic similarity | With suitable pooling/fine-tuning | Possible      | Excellent               |
| RAG retrieval       | Useful variants                   | Not typical   | Very common             |

Examples:

- BERT: `bert-base-uncased`
- Sentence Transformer: `all-MiniLM-L6-v2`
- Embedding model: `text-embedding-004`
- Generative model: Gemini, GPT, Claude.

An important distinction: BERT itself is not automatically a high-quality sentence embedding model. Models such as Sentence-BERT (SBERT) are specifically trained to produce useful sentence-level embeddings.

## **3. Where does BERT fit in LangGraph?**

This is where things become interesting.

**LangGraph is not an AI model. It is an orchestration framework.**

You can integrate BERT models into LangGraph as individual nodes to perform specialized NLP tasks.

For example, imagine you are building an enterprise customer-service agent.

**User Query**

"I want to check my delayed order"

**BERT Intent Classification Node**

BERT model

Identifies customer intent: `ORDER_STATUS`

**BERT NER Node**

Extracts entities such as order ID, date and product.

**LangGraph Conditional Router**

Decides which workflow to execute.

**Order API**

Fetch order details

**RAG**

Retrieve policies

**LLM Response Generation**

Gemini / GPT / Claude

Notice the responsibilities:

- BERT understands and classifies the incoming text.
- LangGraph decides what happens next.
- LLM generates the natural-language response.
- Tools and RAG retrieve actual business information.

## **4. How to integrate BERT into LangGraph**

You can use Hugging Face Transformers to load BERT and invoke it inside a LangGraph node.

First, install dependencies:

```
pip install transformers torch langgraph
```

### **Step 1: Load a BERT classification model**

For illustration, use a pretrained sentiment classifier based on BERT.

```
from transformers import pipelineclassifier = pipeline(    "sentiment-analysis",    model="distilbert-base-uncased-finetuned-sst-2-english")result = classifier(    "I am unhappy with my delayed order")print(result)
```

This returns a sentiment label and confidence score.

Note: DistilBERT is a smaller, distilled BERT-family model.

### **Step 2: Create a LangGraph node**

```
from typing import TypedDictfrom langgraph.graph import StateGraph, START, ENDclass AgentState(TypedDict):    user_query: str    sentiment: str    confidence: floatdef bert_classification_node(state: AgentState):    result = classifier(state["user_query"])[0]    return {        "sentiment": result["label"],        "confidence": result["score"]    }
```

### **Step 3: Build the graph**

```
builder = StateGraph(AgentState)builder.add_node(    "bert_classifier",    bert_classification_node)builder.add_edge(START, "bert_classifier")builder.add_edge("bert_classifier", END)graph = builder.compile()result = graph.invoke({    "user_query": "I am unhappy with my delayed order"})print(result)
```

Now BERT is executing as a node within your LangGraph workflow.

## **5. Important BERT variants you should know**

There are several BERT-family models used in production AI systems.

| Model                 | Typical application                    |
| --------------------- | -------------------------------------- |
| BERT                  | Text understanding and classification  |
| DistilBERT            | Lightweight classification             |
| RoBERTa               | Improved language understanding        |
| ALBERT                | Parameter-efficient language modelling |
| Sentence-BERT (SBERT) | Sentence embeddings and similarity     |
| MiniLM                | Lightweight embeddings and reranking   |
| DeBERTa               | Advanced language understanding        |

Some particularly important applications in RAG:

[Sentence-BERT： Sentence Embeddings using Siamese BERT-Networks | QMMMS's Blog](https://images.openai.com/static-rsc-4/ZfnHZLYQYpYfouoT37Qvq5GQXvqa53uYqlo0xt1Bf76joPNRZHMtHRTDbqHMnyutwoiiazUC-2w1-bOR3jHRdwpJr_pxVyqmj0QtzdcdW2doXehTmLyqgEzCOr-Cv5itd03WR_bnqorTTiWAs-iyUmkS30euBecNKQe18ysvQ_E?purpose=inline)

**Sentence-BERT (SBERT)**

Converts sentences into embeddings for semantic search and similarity comparison.

[My notes from DeepLearning.AI’s course on Advanced Retrieval for AI with Chroma | by Anurag Chatterjee | Medium](https://images.openai.com/static-rsc-4/0hkdAnIVd03znX_LCYxmS2DbIxODWXj2GydS1JKqCgpfEDu0YJZoSR2VmzIMNXwJipJqvTKP21Fs4CtaxbRt2tuB-VDAPr2GJdZwArnebOBh7tlUDVI2qvTJ9Mum4_A4l8snNOOC0QXRkv3IPQbheDREb21KHHVxY_1rhjWL9QE?purpose=inline)

**Cross-Encoder BERT**

Evaluates query-document relevance together. Commonly used for reranking retrieved documents.

[Fundamentals of Natural Language Processing (NLP) in Python: Transformers: BERT and Beyond](https://images.openai.com/static-rsc-4/vC6ZtBetq8DbkvdQx6AK2O90pWiOClfPQPiEroR8USyef1GlTIu9_eUMIot8pO4XOiqGk4Jr0zTGdXGRTjJ9zBnb3_t4VXEZ0c4eUJvchlk833Th57UZMqFNthvX6TWF3hVncqMVigXZXVWspM6FlhqfArnRoRUQ1vyylfx2XdU?purpose=inline)

**BERT NER**

Extracts names, dates, locations, order IDs and other entities from text.

## **6. Where I would use BERT in your production RAG architecture**

| Component                   | Recommended approach               |
| --------------------------- | ---------------------------------- |
| Query intent classification | Fine-tuned BERT                    |
| Query entity extraction     | BERT NER                           |
| Semantic embedding          | SBERT or dedicated embedding model |
| Document reranking          | BERT Cross-Encoder                 |
| Query classification        | BERT / DistilBERT                  |
| Response generation         | Gemini / GPT / Claude              |
| Workflow orchestration      | LangGraph                          |

One important production consideration: don't automatically introduce BERT for every task. If an embedding model, LLM or existing classifier already performs the job adequately, another model introduces additional latency, infrastructure and operational complexity.

## **7. Important distinction to remember**

**Think of these as three different layers:**

**BERT / GPT / Gemini**

The intelligence layer – understands, classifies or generates.

**LangChain**

The integration layer – connects models, prompts, retrievers and tools.

**LangGraph**

The orchestration layer – manages state, nodes, routing, retries, workflows and human approval.

**My recommendation:** For your production AI/RAG architecture learning, study these four topics next:

1. BERT architecture and self-attention.
2. Sentence-BERT vs traditional BERT.
3. Cross-Encoder vs Bi-Encoder and how reranking works.
4. Integrating Hugging Face models into LangGraph nodes.

I can also walk you through building a production-style LangGraph pipeline combining BERT intent classification, SBERT embeddings, a Cross-Encoder reranker and Gemini response generation.

For continued learning, you could also have a fresh practical AI architecture exercise prepared each day.

Daily AI architecture exercise