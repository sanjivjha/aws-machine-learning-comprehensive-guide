# End-to-End Sentiment Analysis Pipeline on AWS

This case study demonstrates a complete workflow for sentiment analysis using AWS services. We'll analyze customer reviews to determine sentiment and visualize the results.

## Prerequisites

- An AWS account with appropriate permissions
- AWS CLI configured with your credentials
- Python 3.7 or later
- Required Python packages: boto3, pandas, numpy

Install required packages:
```bash
pip install boto3 pandas numpy
```

## Step 1: Data Preparation

First, we'll create a sample dataset of customer reviews and upload it to S3. In a real-world scenario, you would replace this with your own data.

```python
import pandas as pd
import numpy as np
import boto3
import io

# Create sample data
np.random.seed(42)
n_reviews = 1000

products = ['Laptop', 'Smartphone', 'Headphones', 'Smartwatch', 'Tablet']
reviews = [
    "This product is amazing! I love it.",
    "Not bad, but could be better.",
    "Terrible experience. Would not recommend.",
    "Great value for money.",
    "It's okay, nothing special."
]

data = pd.DataFrame({
    'review_id': range(1, n_reviews + 1),
    'product': np.random.choice(products, n_reviews),
    'rating': np.random.randint(1, 6, n_reviews),
    'review_text': np.random.choice(reviews, n_reviews)
})

# Upload to S3
bucket_name = 'your-bucket-name'  # Replace with your S3 bucket name
key = 'reviews_data/customer_reviews.csv'

s3 = boto3.client('s3')
csv_buffer = io.StringIO()
data.to_csv(csv_buffer, index=False)
s3.put_object(Bucket=bucket_name, Key=key, Body=csv_buffer.getvalue())

print(f"Data uploaded to s3://{bucket_name}/{key}")

# To use your own data, comment out the code above and use:
# s3.upload_file('path/to/your/data.csv', bucket_name, key)
```

## Step 2: Sentiment Analysis with Amazon Comprehend

We'll use Amazon Comprehend to perform sentiment analysis on our customer reviews.

```python
import boto3
import json
from botocore.exceptions import ClientError

comprehend = boto3.client('comprehend')
s3 = boto3.client('s3')

def analyze_sentiment(text):
    try:
        response = comprehend.detect_sentiment(Text=text, LanguageCode='en')
        return response['Sentiment'], response['SentimentScore']
    except ClientError as e:
        print(f"Error detecting sentiment: {e}")
        return None, None

def process_reviews(bucket, key):
    # Read the CSV file from S3
    obj = s3.get_object(Bucket=bucket, Key=key)
    df = pd.read_csv(io.BytesIO(obj['Body'].read()))
    
    results = []
    for _, row in df.iterrows():
        sentiment, scores = analyze_sentiment(row['review_text'])
        if sentiment is not None:
            results.append({
                'review_id': row['review_id'],
                'product': row['product'],
                'rating': row['rating'],
                'sentiment': sentiment,
                'positive_score': scores['Positive'],
                'negative_score': scores['Negative'],
                'neutral_score': scores['Neutral'],
                'mixed_score': scores['Mixed']
            })
    
    return pd.DataFrame(results)

# Process the reviews
results_df = process_reviews(bucket_name, key)

# Save results to S3
output_key = 'sentiment_results/sentiment_analysis.csv'
csv_buffer = io.StringIO()
results_df.to_csv(csv_buffer, index=False)
s3.put_object(Bucket=bucket_name, Key=output_key, Body=csv_buffer.getvalue())

print(f"Sentiment analysis results saved to s3://{bucket_name}/{output_key}")
```

## Step 3: Data Analysis and Preparation for Visualization

Let's perform some basic analysis on our sentiment results and prepare the data for visualization.

```python
import matplotlib.pyplot as plt

# Read the sentiment analysis results
obj = s3.get_object(Bucket=bucket_name, Key=output_key)
results_df = pd.read_csv(io.BytesIO(obj['Body'].read()))

# Calculate average sentiment scores by product
avg_sentiment = results_df.groupby('product').agg({
    'positive_score': 'mean',
    'negative_score': 'mean',
    'neutral_score': 'mean',
    'mixed_score': 'mean'
}).reset_index()

# Calculate sentiment distribution
sentiment_dist = results_df['sentiment'].value_counts(normalize=True).reset_index()
sentiment_dist.columns = ['sentiment', 'percentage']
sentiment_dist['percentage'] = sentiment_dist['percentage'] * 100

# Save processed data for QuickSight
processed_data_key = 'quicksight_data/processed_sentiment_data.csv'
csv_buffer = io.StringIO()
results_df.to_csv(csv_buffer, index=False)
s3.put_object(Bucket=bucket_name, Key=processed_data_key, Body=csv_buffer.getvalue())

print(f"Processed data for QuickSight saved to s3://{bucket_name}/{processed_data_key}")

# Create a simple visualization
plt.figure(figsize=(10, 6))
sentiment_dist.plot(kind='bar', x='sentiment', y='percentage')
plt.title('Sentiment Distribution')
plt.xlabel('Sentiment')
plt.ylabel('Percentage')
plt.tight_layout()

# Save plot to S3
img_data = io.BytesIO()
plt.savefig(img_data, format='png')
img_data.seek(0)
s3.put_object(Body=img_data, Bucket=bucket_name, Key='sentiment_analysis/sentiment_distribution.png')

print(f"Sentiment distribution plot saved to s3://{bucket_name}/sentiment_analysis/sentiment_distribution.png")
```

## Step 4: Setting up Amazon QuickSight

To visualize the data in QuickSight, follow these steps:

1. Sign in to the AWS Management Console and open the Amazon QuickSight console.

2. If you haven't used QuickSight before, you'll need to sign up for it and grant it permissions to access your S3 bucket.

3. Once in QuickSight, create a new dataset:
   - Choose "New dataset" and select "S3" as the data source.
   - Enter a data source name (e.g., "SentimentAnalysisResults").
   - For the S3 URL, enter: `s3://{bucket_name}/{processed_data_key}` (replace with your actual bucket and key).
   - Choose "Upload" to upload a manifest file, which QuickSight will create for you.

4. After creating the dataset, you can create a new analysis:
   - Click "New analysis" and select your dataset.
   - Use the visual types and fields to create visualizations. For example:
     - Create a pie chart of sentiment distribution
     - Create a bar chart of average sentiment scores by product
     - Create a scatter plot of rating vs. positive sentiment score

5. Save your analysis and optionally publish it as a dashboard for easier sharing.

## Step 5: Automating the Pipeline

To automate this pipeline, you can create an AWS Lambda function that triggers on new data uploads to your S3 bucket. Here's a sample Lambda function:

```python
import boto3
import pandas as pd
import io

s3 = boto3.client('s3')
comprehend = boto3.client('comprehend')

def lambda_handler(event, context):
    # Get the S3 bucket and key from the event
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = event['Records'][0]['s3']['object']['key']
    
    # Process the reviews
    results_df = process_reviews(bucket, key)
    
    # Save results to S3
    output_key = f"sentiment_results/{key.split('/')[-1].replace('.csv', '_analyzed.csv')}"
    csv_buffer = io.StringIO()
    results_df.to_csv(csv_buffer, index=False)
    s3.put_object(Bucket=bucket, Key=output_key, Body=csv_buffer.getvalue())
    
    return {
        'statusCode': 200,
        'body': f"Sentiment analysis completed. Results saved to s3://{bucket}/{output_key}"
    }

def analyze_sentiment(text):
    response = comprehend.detect_sentiment(Text=text, LanguageCode='en')
    return response['Sentiment'], response['SentimentScore']

def process_reviews(bucket, key):
    # Read the CSV file from S3
    obj = s3.get_object(Bucket=bucket, Key=key)
    df = pd.read_csv(io.BytesIO(obj['Body'].read()))
    
    results = []
    for _, row in df.iterrows():
        sentiment, scores = analyze_sentiment(row['review_text'])
        results.append({
            'review_id': row['review_id'],
            'product': row['product'],
            'rating': row['rating'],
            'sentiment': sentiment,
            'positive_score': scores['Positive'],
            'negative_score': scores['Negative'],
            'neutral_score': scores['Neutral'],
            'mixed_score': scores['Mixed']
        })
    
    return pd.DataFrame(results)
```

To set up this Lambda function:

1. Create a new Lambda function in the AWS Console.
2. Copy the code above into the function.
3. Set up an S3 trigger for your input bucket and prefix.
4. Make sure the Lambda function has the necessary permissions to access S3 and Comprehend.

## Conclusion

This end-to-end example demonstrates how to:

1. Prepare and upload customer review data to S3
2. Perform sentiment analysis using Amazon Comprehend
3. Process and analyze the sentiment results
4. Visualize the results using Amazon QuickSight
5. Automate the pipeline using AWS Lambda

Key points to remember:

- Replace the sample data with your own dataset for real-world applications
- Ensure proper error handling and logging for production environments
- Set up monitoring for the Lambda function to track its performance
- Regularly update and retrain your sentiment analysis model if using a custom model
- Consider using Amazon Athena for more complex SQL-based analysis of your results

By following this workflow, you can build and deploy a scalable sentiment analysis pipeline on AWS, leveraging managed services to minimize operational overhead and focus on deriving insights from your data.
