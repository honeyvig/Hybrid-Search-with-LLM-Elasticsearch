# Hybrid-Search-with-LLM-Elasticsearch
Build hybrid search experience that returns knowledge results from the Web and relevant listings from UniversityTech.io utilising LLM/SLM. This can take the form of a chatbot.

Phase 1: Proof of Concept (POC) with LLM + Elasticsearch
Objective: Build a functional search tool using a pre-trained Large Language Model (LLM) combined with Elasticsearch for retrieving relevant listings from the 700 available articles (~60,000 words). Data is growing at 100 articles per month and needs to be synced with Google Sheets.

Components:
LLM Integration: Use an existing LLM (like GPT-3.5 or similar) for processing user queries and generating semantic interpretations.

Elasticsearch: Index the 700 articles with Elasticsearch to power keyword-based search. Use LLM to rank the relevance of results based on semantic understanding.

Basic UI: Develop a minimal, functional user interface allowing users to input queries and receive results from the Elasticsearch database.

Outcome:
Demonstrate improved search capabilities beyond keyword matching by incorporating LLM for query processing and semantic understanding. This POC will provide quick, relevant results, allowing users to search the limited dataset effectively.

1. Simple query example:
"How to improve battery storage efficiency?"
Knowledge result: Explanation of methods to improve battery storage capacity and efficiency.
UniversityTech.io result: Listings related to battery storage improvements, including technologies for energy density enhancement, thermal management systems, or alternative materials for battery components.

2. Complex query example:
"Methods to detect microplastics in ocean water and convert them into useful products?"
Knowledge result: Techniques for detecting and filtering microplastics in marine environments, as well as upcycling methods for plastic waste.
UniversityTech.io result: A combination of technologies for microplastic detection and removal, and others focused on converting recovered plastics into valuable materials, such as upcycled products or biodegradable alternatives.

=================
Python script to build a hybrid search experience that integrates LLM for semantic query processing and Elasticsearch for indexing and searching the articles. The script provides the foundation for a Proof of Concept (POC) as described.
Python Script: Hybrid Search with LLM + Elasticsearch
Requirements:

    Python Libraries:
        openai (for LLM queries, e.g., GPT-3.5)
        elasticsearch (for Elasticsearch operations)
        flask (for building a basic web interface)
        pandas (for managing article data)
        google-auth, gspread (for syncing with Google Sheets)

    Elasticsearch Setup:
        Install Elasticsearch locally or use a managed cloud Elasticsearch service.
        Create an index to store and search articles.

    Google Sheets Integration:
        Set up a Google Service Account and connect it to a Google Sheet for syncing data.

Code Implementation

import openai
from elasticsearch import Elasticsearch
from flask import Flask, request, jsonify
import pandas as pd
import gspread
from oauth2client.service_account import ServiceAccountCredentials

# Configure OpenAI API Key
openai.api_key = "YOUR_OPENAI_API_KEY"

# Elasticsearch Configuration
es = Elasticsearch([{'host': 'localhost', 'port': 9200}])  # Replace with your Elasticsearch host/port

# Flask App Setup
app = Flask(__name__)

# Google Sheets Integration
def sync_with_google_sheets(sheet_id, credentials_file):
    """
    Syncs data from a Google Sheet to a DataFrame.
    """
    scope = ["https://spreadsheets.google.com/feeds", "https://www.googleapis.com/auth/drive"]
    creds = ServiceAccountCredentials.from_json_keyfile_name(credentials_file, scope)
    client = gspread.authorize(creds)
    sheet = client.open_by_key(sheet_id)
    worksheet = sheet.get_worksheet(0)
    data = worksheet.get_all_records()
    return pd.DataFrame(data)

# Sync Data and Index in Elasticsearch
def index_articles(data):
    """
    Indexes articles into Elasticsearch.
    """
    index_name = "universitytech_articles"

    # Delete existing index if exists
    if es.indices.exists(index=index_name):
        es.indices.delete(index=index_name)

    # Create a new index
    es.indices.create(index=index_name)

    # Index each article
    for i, article in data.iterrows():
        doc = {
            "title": article['Title'],
            "content": article['Content'],
            "tags": article['Tags']
        }
        es.index(index=index_name, id=i, body=doc)

    print(f"Indexed {len(data)} articles.")

# Semantic Query Processing with LLM
def semantic_query(query):
    """
    Processes user query using LLM for semantic interpretation.
    """
    response = openai.ChatCompletion.create(
        model="gpt-3.5-turbo",
        messages=[
            {"role": "system", "content": "You are a helpful assistant for semantic search."},
            {"role": "user", "content": query}
        ]
    )
    return response['choices'][0]['message']['content']

# Elasticsearch Search
def search_articles(query, semantic_context=""):
    """
    Searches articles in Elasticsearch using a combination of keywords and semantic context.
    """
    index_name = "universitytech_articles"
    search_body = {
        "query": {
            "bool": {
                "must": [
                    {"match": {"content": query}},
                ],
                "should": [
                    {"match": {"content": semantic_context}}
                ]
            }
        }
    }
    res = es.search(index=index_name, body=search_body)
    return res['hits']['hits']

# Flask Route for Search
@app.route("/search", methods=["POST"])
def search():
    """
    API Endpoint to handle search requests.
    """
    user_query = request.json.get("query")
    
    # Get semantic interpretation of the query
    semantic_context = semantic_query(user_query)
    
    # Search in Elasticsearch
    results = search_articles(user_query, semantic_context)
    
    # Combine results
    response = {
        "query": user_query,
        "semantic_context": semantic_context,
        "results": [
            {
                "title": result["_source"]["title"],
                "content": result["_source"]["content"],
                "score": result["_score"]
            }
            for result in results
        ]
    }
    return jsonify(response)

# Main Execution
if __name__ == "__main__":
    # Sync with Google Sheets
    google_sheet_id = "YOUR_GOOGLE_SHEET_ID"
    credentials_file = "path_to_your_service_account_credentials.json"
    articles_df = sync_with_google_sheets(google_sheet_id, credentials_file)

    # Index articles in Elasticsearch
    index_articles(articles_df)

    # Run Flask App
    app.run(host="0.0.0.0", port=5000)

Features and Workflow

    Data Synchronization:
        Sync article data from Google Sheets into a Pandas DataFrame.
        Data includes fields such as Title, Content, and Tags.

    Elasticsearch:
        Stores and indexes articles.
        Supports keyword-based search and ranking using semantic contexts from LLM.

    LLM Integration:
        Processes user queries to extract semantic meaning.
        Uses the result to enhance Elasticsearch query relevance.

    API:
        Exposes a /search endpoint for sending queries via POST requests.
        Combines Elasticsearch results with semantic enhancements.

    Basic UI:
        Can be built on top of the Flask API using HTML/JavaScript for a minimal web interface.

Example Query and Response
Query:

{
    "query": "Methods to improve battery efficiency in renewable energy storage"
}

Response:

{
    "query": "Methods to improve battery efficiency in renewable energy storage",
    "semantic_context": "Ways to enhance battery performance, such as energy density, thermal management, and alternative materials.",
    "results": [
        {
            "title": "Enhancing Battery Storage for Renewable Energy",
            "content": "This article discusses various methods to improve battery storage, including better materials and management systems.",
            "score": 1.23
        },
        {
            "title": "Energy Density Optimization in Batteries",
            "content": "Explores techniques to increase energy density and extend battery life for renewable energy applications.",
            "score": 1.18
        }
    ]
}

Next Steps

    Deploy Elasticsearch:
        Use Elastic Cloud or AWS Elasticsearch for scalability.
    Enhance LLM Integration:
        Fine-tune GPT models on domain-specific data for better contextual understanding.
    Develop a UI:
        Build a simple React or Vue.js front-end to interact with the search API.
    Scale for Real-Time Updates:
        Automate Google Sheets sync to update Elasticsearch indices.

This script delivers the essential components for the POC and can be extended for production use.
