
# Chapter 8: Generative AI on AWS

## 8.1 Introduction to Generative AI

Generative AI refers to artificial intelligence systems that can create new content, including text, images, code, and more. AWS offers several services and tools to leverage generative AI capabilities, enabling developers to build sophisticated AI-powered applications.

## 8.2 Amazon Bedrock

Amazon Bedrock is a fully managed service that provides access to high-performing foundation models (FMs) from leading AI companies through a single API.

**Key Features:**

- Access to various foundation models (e.g., Claude from Anthropic, Jurassic-2 from AI21 Labs)
- Customization capabilities through fine-tuning
- Serverless experience with pay-as-you-go pricing
- Enterprise-grade security and privacy

**Example: Text Generation with Bedrock**

```python
import boto3
import json

bedrock = boto3.client(service_name='bedrock-runtime')

prompt = "Write a short story about a robot learning to paint:"

body = json.dumps({
    "prompt": prompt,
    "max_tokens_to_sample": 300,
    "temperature": 0.7,
    "top_p": 0.8,
})

modelId = 'anthropic.claude-v2'
accept = 'application/json'
contentType = 'application/json'

response = bedrock.invoke_model(body=body, modelId=modelId, accept=accept, contentType=contentType)
response_body = json.loads(response.get('body').read())

print(response_body.get('completion'))
```

## 8.3 Amazon SageMaker JumpStart

SageMaker JumpStart provides pre-trained, open-source models for a wide variety of problem types to help you get started with machine learning.

**Key Features:**

- One-click deployment of pre-trained models
- Fine-tuning capabilities
- Integration with SageMaker's managed infrastructure

**Example: Deploying a JumpStart Model**

```python
import sagemaker
from sagemaker import get_execution_role
from sagemaker.jumpstart.model import JumpStartModel

role = get_execution_role()
region = sagemaker.Session().boto_region_name

model_id = "huggingface-text2text-flan-t5-xl"
model = JumpStartModel(model_id=model_id)

predictor = model.deploy()

result = predictor.predict("Translate to French: Hello, how are you?")
print(result)

predictor.delete_endpoint()
```

## 8.4 Retrieval-Augmented Generation (RAG) Application

RAG combines the power of large language models with a retrieval system to generate more accurate and contextually relevant responses. Let's build a RAG application using Amazon Bedrock and Amazon Kendra.

### 8.4.1 Setting Up the Environment

```python
import boto3
import requests
import json
import time
import os

s3 = boto3.client('s3')
kendra = boto3.client('kendra')
bedrock = boto3.client('bedrock-runtime')

bucket_name = 'your-bucket-name'
kendra_role_arn = 'arn:aws:iam::your-account-id:role/KendraRoleWithRequiredPermissions'
s3_access_role_arn = 'arn:aws:iam::your-account-id:role/KendraRoleWithS3Access'
```

### 8.4.2 Indexing Documents with Kendra

```python
def download_and_upload_pdf():
    pdf_url = "https://docs.aws.amazon.com/pdfs/whitepapers/latest/aws-overview/aws-overview.pdf"
    pdf_name = "aws-overview.pdf"
    
    response = requests.get(pdf_url)
    with open(pdf_name, 'wb') as f:
        f.write(response.content)
    
    s3.upload_file(pdf_name, bucket_name, f"documents/{pdf_name}")
    print(f"AWS Overview PDF uploaded to s3://{bucket_name}/documents/{pdf_name}")
    
    os.remove(pdf_name)

def create_kendra_index():
    response = kendra.create_index(
        Name='AWSOverviewIndex',
        Edition='DEVELOPER_EDITION',
        RoleArn=kendra_role_arn
    )
    index_id = response['Id']
    
    while True:
        response = kendra.describe_index(Id=index_id)
        if response['Status'] == 'ACTIVE':
            break
        time.sleep(60)
    
    print(f"Kendra Index created with ID: {index_id}")
    return index_id

def create_kendra_datasource(index_id):
    response = kendra.create_data_source(
        IndexId=index_id,
        Name='AWSOverviewDataSource',
        Type='S3',
        DataSourceConfiguration={
            'S3Configuration': {
                'BucketName': bucket_name,
                'InclusionPrefixes': ['documents/aws-overview.pdf']
            }
        },
        RoleArn=s3_access_role_arn
    )
    data_source_id = response['Id']
    
    kendra.start_data_source_sync_job(Id=data_source_id, IndexId=index_id)
    
    while True:
        sync_status = kendra.describe_data_source(Id=data_source_id, IndexId=index_id)
        if sync_status['Status'] == 'ACTIVE':
            break
        time.sleep(60)
    
    print("AWS Overview PDF indexed in Kendra")
    return data_source_id

# Execute the setup
download_and_upload_pdf()
index_id = create_kendra_index()
data_source_id = create_kendra_datasource(index_id)
```

### 8.4.3 Implementing the RAG Application

```python
def query_kendra(query, index_id):
    response = kendra.query(
        IndexId=index_id,
        QueryText=query
    )
    return response['ResultItems']

def generate_bedrock_response(query, context):
    prompt = f"""Human: You are an AI assistant with knowledge about AWS services. Use the following information from the AWS Overview whitepaper to answer the human's question. If the information doesn't contain the answer, say you don't know but provide general information about AWS if relevant.

Information:
{context}
```
To use the RAG application and chat with it, you would typically create a function that combines the Kendra querying and Bedrock response generation. Here's an example of how you might implement this:

```python
def rag_chat(query, index_id):
    # Query Kendra for relevant information
    kendra_results = query_kendra(query, index_id)
    
    # Extract and combine the relevant text from Kendra results
    context = "\n".join([result['DocumentExcerpt']['Text'] for result in kendra_results[:3]])
    
    # Generate a response using Bedrock
    bedrock_response = generate_bedrock_response(query, context)
    
    return bedrock_response

# Example usage
index_id = "your-kendra-index-id"  # Replace with your actual Kendra index ID

while True:
    user_input = input("You: ")
    if user_input.lower() in ['exit', 'quit', 'bye']:
        print("AI: Goodbye! Have a great day!")
        break
    
    response = rag_chat(user_input, index_id)
    print("AI:", response)
```

In this implementation:

1. The `rag_chat` function takes a user query and the Kendra index ID as input.
2. It first queries Kendra to retrieve relevant information from the indexed documents.
3. It then extracts the text from the top 3 Kendra results and combines them into a context string.
4. This context, along with the original query, is sent to the Bedrock model to generate a response.
5. The generated response is returned.

The example usage shows how you might set up a simple chat loop:

1. It continually prompts the user for input.
2. It sends each user input to the `rag_chat` function.
3. It prints the AI's response.
4. The loop continues until the user types 'exit', 'quit', or 'bye'.

To use this RAG application, you would run this script and start chatting. The AI would use the information from your indexed documents (in this case, the AWS Overview whitepaper) to inform its responses, providing more accurate and contextually relevant information about AWS services.

Remember to replace "your-kendra-index-id" with the actual ID of the Kendra index you created earlier in the setup process.

This implementation allows for a conversational interface with the RAG system, where users can ask questions and receive responses based on the indexed document and the capabilities of the language model.


## 8.5 Comparing SageMaker JumpStart and Amazon Bedrock

### SageMaker JumpStart

SageMaker JumpStart provides a wide variety of pre-trained models that can be easily deployed or fine-tuned.

Sample list of models available through JumpStart:

1. Text Processing:
   - BERT (various versions)
   - RoBERTa
   - DistilBERT
   - XLM
2. Image Processing:
   - ResNet (various versions)
   - MobileNet
   - YOLOv5
3. Tabular Data:
   - XGBoost
   - LightGBM
   - CatBoost
4. Generative AI:
   - GPT-2
   - T5

### Amazon Bedrock

Bedrock provides access to foundation models from leading AI companies.

Sample list of models available through Bedrock:

1. Anthropic:
   - Claude (various versions)
2. AI21 Labs:
   - Jurassic-2 (various versions)
3. Amazon:
   - Titan (various versions)
4. Stability AI:
   - Stable Diffusion (for image generation)

### Key Differences

1. Model Source:
   - JumpStart: Mostly open-source models
   - Bedrock: Proprietary models from AI companies
2. Customization:
   - JumpStart: Allows fine-tuning of models
   - Bedrock: Offers customization through prompt engineering and fine-tuning (for some models)
3. Deployment:
   - JumpStart: Models are deployed to SageMaker endpoints
   - Bedrock: Serverless access to models through API calls
4. Use Cases:
   - JumpStart: Wide range of ML tasks (classification, regression, NLP, computer vision)
   - Bedrock: Primarily focused on large language models and generative AI
5. Integration:
   - JumpStart: Tightly integrated with SageMaker ecosystem
   - Bedrock: Can be easily integrated into any application

### When to Use Each

Use SageMaker JumpStart when:
- You need a wide variety of ML models for different tasks
- You want to fine-tune models on your specific dataset
- You're already using the SageMaker ecosystem
- You need more control over the deployment infrastructure

Use Amazon Bedrock when:
- You need state-of-the-art large language models
- You want a serverless solution with minimal infrastructure management
- Your use case requires the latest generative AI capabilities
- You need enterprise-grade security and compliance features

In many cases, you might use both services in your AI/ML pipeline. For example, you could use JumpStart for data preprocessing or feature extraction, and then use Bedrock for generating human-like text based on those features.

## 8.6 Example: Combining JumpStart and Bedrock in a Pipeline

Here's a simple example of how you might combine JumpStart and Bedrock in a pipeline:

```python
import sagemaker
from sagemaker.jumpstart.model import JumpStartModel
import boto3
import json

# Set up SageMaker session
sagemaker_session = sagemaker.Session()
role = sagemaker.get_execution_role()

# Deploy a JumpStart model for sentiment analysis
sentiment_model = JumpStartModel(model_id="huggingface-sentiment-distilbert-base-uncased-finetuned-sst-2-english")
sentiment_predictor = sentiment_model.deploy()

# Set up Bedrock client
bedrock = boto3.client(service_name='bedrock-runtime')

def analyze_and_respond(user_input):
    # Analyze sentiment using JumpStart model
    sentiment = sentiment_predictor.predict({"inputs": user_input})
    sentiment_label = "positive" if sentiment[0]['label'] == "LABEL_1" else "negative"

    # Generate response using Bedrock
    prompt = f"""Human: The user's message "{user_input}" has been analyzed as having a {sentiment_label} sentiment. 
    Please generate a response that acknowledges this sentiment and provides a helpful reply.

    Assistant: Certainly! Here's a response that acknowledges the sentiment and provides a helpful reply:

    {sentiment_label.capitalize()} sentiment detected. I understand that you're feeling {sentiment_label} about this. 

    Human: Great, now please provide the actual response to the user based on their input and the detected sentiment.

    Assistant: Certainly! Here's a response tailored to the user's input and detected sentiment:

    """

    body = json.dumps({
        "prompt": prompt,
        "max_tokens_to_sample": 200,
        "temperature": 0.7,
        "top_p": 0.8,
    })

    modelId = 'anthropic.claude-v2'
    response = bedrock.invoke_model(body=body, modelId=modelId)
    response_body = json.loads(response.get('body').read())

    return response_body.get('completion').strip()

# Example usage
user_message = "I'm really excited about learning AWS services!"
response = analyze_and_respond(user_message)
print(f"User: {user_message}")
print(f"AI: {response}")

# Clean up
sentiment_predictor.delete_endpoint()
```

This example demonstrates how you can use a JumpStart model for sentiment analysis and then use that information to generate a more contextually appropriate response with a Bedrock model.

By combining these services, you can create sophisticated AI applications that leverage the strengths of both SageMaker JumpStart and Amazon Bedrock.
