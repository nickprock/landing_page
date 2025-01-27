---
title: Agentic RAG Discord Bot with CAMEL-AI
weight: 14
partition: build
social_preview_image: /documentation/examples/agentic-rag-camelai-discord/social-preview.png
---

![agentic-rag-camelai-astronaut](/documentation/examples/agentic-rag-camelai-discord/astronaut-main.png)

# Agentic RAG Discord ChatBot with Qdrant, CAMEL-AI, & OpenAI

| Time: 45 min | Level: Intermediate | [![Open in Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/drive/1Ymqzm6ySoyVOekY7fteQBCFCXYiYyHxw#scrollTo=QQZXwzqmNfaS) |
| --- | ----------- | ----------- |----------- |




Unlike traditional RAG techniques, which passively retrieve context and generate responses, **agentic RAG** involves active decision-making and multi-step reasoning by the chatbot. This approach allows the bot to dynamically interact with various data sources, adapt its behavior based on context, and perform more complex tasks autonomously.

In this tutorial, we’ll develop a Discord chatbot using agentic RAG principles. The bot will use:

- Qdrant for efficient vector search,
- [CAMEL-AI](https://www.camel-ai.org/) for dialogue management, and
- OpenAI models for generating embeddings and responses.

You'll learn how to set up the environment, scrape and prepare data, and deploy a fully functional chatbot on Discord.
Let’s get started!

---

## Workflow Overview

Below is a high-level look at our Agentic RAG workflow:


| Step                  | Description                                                                                              |
|-----------------------|----------------------------------------------------------------------------------------------------------|
| 1. Environment Setup  | Install necessary libraries and dependencies.                                                           |
| 2. Qdrant Configuration | Create a Qdrant Cloud account, set up a cluster, and connect using the API key and cluster URL.         |
| 3. Data Scraping      | Scrape relevant documentation from Qdrant's website for knowledge base creation.                        |
| 4. Chunking & Embedding | Chunk large texts and generate embeddings using OpenAI.                                                 |
| 5. Vector Store Creation | Create and populate a Qdrant collection with the generated embeddings.                                  |
| 6. Context Retrieval  | Define a function to retrieve context from Qdrant based on user queries.                                |
| 7. Discord Bot Setup  | Configure a new Discord bot, invite it to a server, and grant necessary permissions.                    |
| 8. Bot Integration    | Integrate the bot with Qdrant and CAMEL-AI to handle user interactions and provide responses.           |
| 9. Testing            | Test the bot in a live Discord server.                                                                 |


## Architecture Diagram

Below is the architecture diagram representing the workflow and interactions of the chatbot:

![Architecture Diagram](/documentation/examples/agentic-rag-camelai-discord/architecture-diagram.jpg)

The workflow starts with data ingestion. HTML documents are scraped using BeautifulSoup to extract text content, forming the knowledge base for the system.

The embeddings and metadata are stored in a Qdrant collection for structured storage and retrieval. When a user sends a query through the Discord bot, CAMEL-AI's Qdrant Storage Class interfaces with Qdrant to retrieve relevant vectors based on the query.

The retrieved vectors are processed by an AI agent using OpenAI's language model. The AI agent generates a response that is contextually relevant to the user's query. This response is then delivered back to the user through the Discord bot interface, completing the flow.

---

## **Step 1: Environment Setup**

Before diving into the implementation, here's a high-level overview of the stack we'll use:

| **Component**   | **Purpose**                                                                                           |
|-----------------|-------------------------------------------------------------------------------------------------------|
| **Qdrant**      | Vector database for storing and querying document embeddings.                                         |
| **OpenAI**   | Embedding and language model for generating vector representations and chatbot responses.                       |
| **CAMEL-AI**    | Dialogue management framework that powers the chatbot's reasoning and interactions.                   |
| **Discord API** | Platform for deploying and interacting with the chatbot.                                              |

### Install Dependencies

To build our chatbot, we'll need a set of core libraries for embedding generation, vector storage, web scraping, and interacting with the Discord API.

Below is the command to install all necessary dependencies:

```python
!pip install openai qdrant-client camel-ai[all]==0.2.16 requests beautifulsoup4 nest_asyncio discord.py tqdm python-dotenv
```

### Dependency Breakdown

Here’s a quick explanation of what each dependency does:

- `openai`: Generates embeddings and chatbot responses.
- `qdrant-client`: Connects to and interacts with the Qdrant.
- `camel-ai`: Provides multi-agent dialogue management tools.
- `beautifulsoup4` and `requests`: Facilitate web scraping.
- `nest_asyncio`: Allows nested event loops, required for running asynchronous Discord bots.
- `discord.py`: Enables bot interaction with Discord.
- `python-dotenv`: Manages API keys securely.

---

### Set Up OpenAI Client

1. **Create an OpenAI Account**: Go to [OpenAI](https://platform.openai.com/signup) and sign up for an account if you don’t already have one.

2. **Generate an API Key**:

    - After logging in, click on your profile icon in the top-right corner and select **API keys**.

    - Click **Create new secret key**.

    - Copy the generated API key and store it securely. You won’t be able to see it again.

Here’s how to set up the OpenAI client in your code:

Create a `.env` file in your project directory and add your API key:

```bash
OPENAI_API_KEY=<your_openai_api_key>
```

Make sure to replace <your_openai_api_key> with your actual API key.

Now, start the OpenAI Client

```python
import openai
import os
from dotenv import load_dotenv

load_dotenv()

openai_client = openai.Client(
    api_key=os.getenv("OPENAI_API_KEY")
)
```


## **Step 2: Configure the Qdrant Client**

For this tutorial, we will be using the **Qdrant Cloud Free Tier**. Here's how to set it up:

1. **Create an Account**:  Sign up for a Qdrant Cloud account at [Qdrant Cloud](https://cloud.qdrant.io).

2. **Create a Cluster**:  
   - Navigate to the **Overview** section.  
   - Follow the onboarding instructions under **Create First Cluster** to set up your cluster.  
   - When you create the cluster, you will receive an **API Key**. Copy and securely store it, as you will need it later.  

3. **Wait for the Cluster to Provision**:  
   - Your new cluster will appear under the **Clusters** section.               

After obtaining your Qdrant Cloud details, add to your `.env` file:

```bash
QDRANT_CLOUD_URL=<your-qdrant-cloud-url>
QDRANT_CLOUD_API_KEY=<your-api-key>
```

Connect to your Qdrant Cloud instance:

```python
from qdrant_client import QdrantClient

# Set your Qdrant Cloud details
QDRANT_CLOUD_URL = os.getenv("QDRANT_CLOUD_URL")
QDRANT_CLOUD_API_KEY = os.getenv("QDRANT_CLOUD_API_KEY")

collection_name = "discord-bot"

client = QdrantClient(
    url=QDRANT_CLOUD_URL,
    api_key=QDRANT_CLOUD_API_KEY
)
```
Make sure to update the <your-qdrant-cloud-url> and <your-api-key> fields.

---

## **Step 3: Scrape and Prepare Data**

We'll use BeautifulSoup to scrape content from Qdrant's documentation. The extracted text will be prepared for embedding and later used for querying.

```python
import requests
from bs4 import BeautifulSoup
from tqdm import tqdm

qdrant_urls = [
    "https://qdrant.tech/documentation/overview",
    "https://qdrant.tech/documentation/guides/installation",
    "https://qdrant.tech/documentation/concepts/filtering",
    "https://qdrant.tech/documentation/concepts/indexing",
    "https://qdrant.tech/documentation/guides/distributed_deployment",
    "https://qdrant.tech/documentation/guides/quantization"
    # Add more URLs as needed
]

def scrape_qdrant_pages(urls):
    documents = []
    metadata = []
    for url in tqdm(urls, desc="Scraping URLs"):
        try:
            response = requests.get(url, timeout=10)
            response.raise_for_status()
            soup = BeautifulSoup(response.text, "html.parser")
            text = soup.get_text(separator="\n", strip=True)
            if text:
                documents.append(text)
                metadata.append({"source_url": url})
        except Exception as e:
            print(f"Error scraping {url}: {e}")
    return documents, metadata

all_docs, all_metadata = scrape_qdrant_pages(qdrant_urls)

```

### Chunk Large Texts

Since some of the scraped documents might be large, we need to split them into smaller, manageable chunks before generating embeddings.

```python
def chunk_texts_with_metadata(texts, metadata, max_length=500):
    chunks = []
    chunk_metadata = []
    for doc_index, text in enumerate(texts):
        words = text.split()
        for i in range(0, len(words), max_length):
            chunk = " ".join(words[i:i + max_length])
            chunks.append(chunk)
            chunk_metadata.append(metadata[doc_index])  # Associate metadata with each chunk
    return chunks, chunk_metadata

# Create chunked docs and corresponding metadata
chunked_docs, chunked_metadata = chunk_texts_with_metadata(all_docs, all_metadata)
```

---
### Generate Embeddings Using OpenAI

We’ll use OpenAI’s embedding model to generate vector representations of the chunked documents.

```python
embedding_model = "text-embedding-3-small"

result = openai_client.embeddings.create(input=chunked_docs, model=embedding_model)
```

---
## **Step 4: Add the Data to Qdrant**

Before creating and populating the Qdrant collection, we need to structure the data into points:

```python
from qdrant_client.models import PointStruct

# Create points for Qdrant
points = [
    PointStruct(
        id=idx,  # Unique ID for each point
        vector=data.embedding,  # Access embedding as an attribute
        payload={
            "text": text,  # Use chunked text
            "source_url": chunked_metadata[idx]['source_url']  # Attach corresponding metadata
        },
    )
    for idx, (data, text) in enumerate(zip(result.data, chunked_docs))
]
```

### Create the Collection

We create a Qdrant collection to store the document embeddings.

```python
from qdrant_client.models import VectorParams, Distance

if not client.collection_exists(collection_name):

    client.create_collection(
        collection_name,
        vectors_config=VectorParams(
        size=1536,
        distance=Distance.COSINE,
        ),
    )
```
We use a vector size of 1536 because it's the dimensionality of embeddings produced by OpenAI's model `text-embedding-3-small`.

### Upload the Points

```python
client.upsert(collection_name=collection_name, points=points)
```
---

## **Step 5: Setup the CAMEL-AI Instances**

### Set the Qdrant Storage Instance

The `QdrantStorage` class provides methods for reading from and writing to a Qdrant instance. You can now pass an instance of this class to retrievers to interact with your Qdrant collections.

```python
from camel.storages import QdrantStorage, VectorDBQuery, VectorRecord
from camel.types import VectorDistance

qdrant_storage = QdrantStorage(
    url_and_api_key=(
        QDRANT_CLOUD_URL,
        QDRANT_CLOUD_API_KEY,
    ),
    collection_name=collection_name,
    distance=VectorDistance.COSINE,
    vector_dim=1536,
)
```

### Set the OpenAI Instance

Define the OpenAI model and create a CAMEL-AI compatible OpenAI instance. 

```python
from camel.configs import ChatGPTConfig
from camel.models import ModelFactory
from camel.types import ModelPlatformType, ModelType

# Create a ChatGPT configuration
config = ChatGPTConfig(temperature=0.2).as_dict()

# Create an OpenAI model using the configuration
openai_model = ModelFactory.create(
    model_platform=ModelPlatformType.OPENAI,
    model_type=ModelType.GPT_4O_MINI,
    model_config_dict=config,
)

# Use the created model
model = openai_model

```
## **Step 6: Define the AutoRetriever**

Next, define the let's define the AutoRetriever implementation that handles both embedding and storing data and executing queries.

```python
from camel.retrievers import AutoRetriever
from camel.types import StorageType
from camel.agents import ChatAgent

assistant_sys_msg = """You are a helpful assistant to answer question,
         I will give you the Original Query and Retrieved Context,
        answer the Original Query based on the Retrieved Context,
        if you can't answer the question just say I don't know."""
auto_retriever = AutoRetriever(
              url_and_api_key=(
                QDRANT_CLOUD_URL,
                QDRANT_CLOUD_API_KEY,
              ),
              storage_type=StorageType.QDRANT,
              embedding_model=embedding_model
            )
qdrant_agent = ChatAgent(system_message=assistant_sys_msg, model=model)

```

---

## **Step 7: Create and Configure the Discord Bot**

Now let's bring the bot to life! It will serve as the interface through which users can interact with the agentic RAG system you’ve built.

### Create a New Discord Bot

1. Go to the [Discord Developer Portal](https://discord.com/developers/applications) and log in with your Discord account.

2. Click on the **New Application** button.

3. Give your application a name and click **Create**.

4. Navigate to the **Bot** tab on the left sidebar and click **Add Bot**.

5. Once the bot is created, click **Reset Token** under the **Token** section to generate a new bot token. Copy this token securely as you will need it later.

### Invite the Bot to Your Server

1. Go to the **OAuth2** tab and then to the **URL Generator** section.

2. Under **Scopes**, select **bot**.

3. Under **Bot Permissions**, select the necessary permissions:

   - Send Messages

   - Read Message History

4. Copy the generated URL and paste it into your browser.

5. Select the server where you want to invite the bot and click **Authorize**.

### Grant the Bot Permissions

1. Go back to the **Bot** tab.

2. Enable the following under **Privileged Gateway Intents**:

   - Server Members Intent

   -  Message Content Intent

Now, the bot is ready to be integrated with your code.

## **Step 8: Build the Discord Bot**

Add to your `.env` file:

```bash
DISCORD_BOT_TOKEN=<your-discord-bot-token>
```

We'll use `discord.py` to create a simple Discord bot that interacts with users and retrieves context from Qdrant before responding.

```python
from camel.bots import DiscordApp
import nest_asyncio
import discord

nest_asyncio.apply()
discord_q_bot = DiscordApp(token=os.getenv("DISCORD_BOT_TOKEN"))

@discord_q_bot.client.event # triggers when a message is sent in the channel
async def on_message(message: discord.Message):
    if message.author == discord_q_bot.client.user:
        return

    if message.type != discord.MessageType.default:
        return

    if message.author.bot:
        return
    user_input = message.content

    retrieved_info = auto_retriever.run_vector_retriever(
        query=user_input,
        contents=[
            "https://qdrant.tech/articles/what-is-a-vector-database/",
        ],
        top_k=10,
        similarity_threshold = 0.3,
        return_detailed_info=True,
    )

    user_msg = str(retrieved_info)
    assistant_response = qdrant_agent.step(user_msg)
    response_content = assistant_response.msgs[0].content

    if len(response_content) > 2000: # discord message length limit
        for chunk in [response_content[i:i+2000] for i in range(0, len(response_content), 2000)]:
            await message.channel.send(chunk)
    else:
        await message.channel.send(response_content)

discord_q_bot.run()
```
---

## **Step 9: Test the Bot**

1. Invite your bot to your Discord server using the OAuth2 URL from the Discord Developer Portal.

2. Run the notebook.

3. Start chatting with the bot in your Discord server. It will retrieve context from Qdrant and provide relevant answers based on your queries.

![agentic-rag-discord-bot-what-is-quantization](/documentation/examples/agentic-rag-camelai-discord/example.png)

---


## Conclusion

Great work coming this far! You’ve built an advanced, agentic RAG-powered Discord chatbot that delivers intelligent, context-aware responses in real time. This project combines several modern AI components into a practical, scalable system. Let’s quickly recap the key milestones:

- **Collection-Level Knowledge Retrieval:** With Qdrant’s vector search, the chatbot can pull the most relevant information from large datasets, ensuring clear and helpful responses.

- **High-Quality Embeddings with OpenAI:** Using OpenAI’s embedding model, you turned text into high-dimensional vectors, making it easy for the bot to find and use relevant data.

- **Autonomous Reasoning with CAMEL-AI:** Thanks to CAMEL-AI’s framework, the chatbot uses multi-step reasoning to generate insightful and intelligent answers.

- **Live Discord Deployment:** You launched the chatbot on Discord, making it interactive and ready to help real users.

With the ability to perform efficient retrieval across large collections, you’re now well-equipped to tackle more complex real-world problems that require scalable, autonomous knowledge systems.