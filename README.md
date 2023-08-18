## Introduction

Vector search is a way to find similar data based on its meaning. It uses machine learning to turn data into vectors, which are like high-dimensional fingerprints. These vectors capture the meaning of the data, so that similar vectors can be found by searching through the vector space. Vector search can be used to complement traditional keyword-based search, and it is also being used to improve the accuracy of large language models (LLMs). LLMs are powerful language models that can generate text, translate languages, and answer questions. However, they can sometimes be inaccurate or incomplete. Vector search can be used to provide ground truth to LLMs, which can help them to improve their accuracy. In search use cases, vector search can be used to find relevant results even when the exact wording of the query is not known. For example, if you search for "restaurants near me," vector search can find restaurants that are similar to the ones you have already visited, even if you don't remember the exact names of the restaurants. Vector search can be useful in a variety of contexts, such as natural language processing and recommendation systems. It is a powerful technique that can be used to find similar data based on its meaning.



## Set up

### 1. Create MongoDB Account
To create a MongoDB Atlas account, you need to register to MongoDB using one of the below methods:
* Through the [Google Cloud Marketplace](https://console.cloud.google.com/marketplace/product/mongodb/mdb-atlas-self-service) if you have a Google Cloud account. Refer to the [documentation](https://www.mongodb.com/docs/atlas/billing/gcp-self-serve-marketplace/) to set up your subscription.
* With the [Atlas registration page](https://www.mongodb.com/cloud/atlas/register).

### 2. Create MongoDB Cluster
To set up a MongoDB Atlas cluster on MongoDB Atlas, Sign In to Atlas UI.

* Click on the **Create Cluster** button in the top right corner of your screen.
* Select Cluster type. 
* Select cluster tier. Provide a name your custer and Click on **Create**.


### 3. Create a Database and a Collection
* Navigate to the collections using the **Browse Collection** button.
* Click on Create Database. Let us build vector embeddings on the sample-mflix database.



### 4. Set up Atlas Trigger
Now that you have a cluster, databases and collections set up you can proceed with Triggers. We are using triggers to add embeddings to the document that is being created. The triggers will call the Google vertex AI APIs to build embeddings and an update operation is performed on the document.

* Click on Triggers from Atlas side pane.

* Click on Add Triggers.

* Select Database trigger. 
* Select cluster, database and collection to trigger.
* Select Insert, Update and  as the Operation type.
* Copy the below script and paste it to the Atlas functions. Update the values for <gcp-porject-id> , Database name and collection. And save the trigger.




```

exports = async function(changeEvent) {
  const fetch = require("node-fetch");
  var myHeaders = new fetch.Headers();
  myHeaders.append("Content-Type", "application/json");
  const doc = changeEvent.fullDocument;

  var raw = JSON.stringify(
    doc.raw_data
  );

  var requestOptions = {
    method: 'POST',
    headers: myHeaders,
    body: raw,
    redirect: 'follow'
  };

  const response = await fetch("<google cloud function trigger https url>", requestOptions)
    .catch(error => console.log('error', error));
  const data = await response.json();
  console.log(data);

  
  const mongodb = context.services.get('<cluster name>');
  const db = mongodb.db('<database name>'); 
  const collection = db.collection('<collection name>'); 
  
  const result = await collection.updateOne({ _id: doc._id },{ $set: { embedding_data: data.embeddings[0] }});

  if(result.modifiedCount === 1) {
      console.log("Successfully updated the document.");
  } else {
      console.log("Failed to update the document.");
  }
};
```



### 5. Create a Google cloud function:

We will use the Google cloud function to generate the text embeddings using Vertex AI APIs.
* To create a Google cloud function, Navigate to [Google Cloud Functions](https://console.cloud.google.com/functions/). 
* Click on Create function button.

```

import functions_framework
from vertexai.language_models import TextEmbeddingModel


@functions_framework.http
def hello_http(request):
    model = TextEmbeddingModel.from_pretrained("textembedding-gecko@001")
    
    request_json = request.get_json(silent=True)
    embeddings = model.get_embeddings([request_json])
    print(embeddings)
    vector = []
    for embedding in embeddings:
        vector.append(embedding.values)
    print(vector)
    return {"embeddings": vector}
    
```

### 6. Create Atlas Vector Search Index
Head over to Atlas search by Navigating to the collection. We are building a search index on the company-catalog collection in this demo.
Click on Create Index. 
Select JSON Editor as configuration Method. 
select your Database and Collection on the left and a drop in the code snippet below for your index definition.

```
{
  "mappings": {
    "dynamic": true,
    "fields": {
      "embedding_data": {
        "dimensions":768,
        "similarity": "euclidean",
        "type": "knnVector"
      }
    }
  }
}
```

Click on Create Index.



### Search text Embeddings
Run the below code to fetch the data from MongoDB using Vector search. 

```
var axios = require('axios');
const MongoClient = require('mongodb').MongoClient;

async function main(){
    var data = JSON.stringify({"instances": [{"content": "Query"}]}); // replace the Query with your data to be searched

    var config = {
      method: 'post',
      url: 'https://us-central1-aiplatform.googleapis.com/v1/projects/<project-id>/locations/us-central1/publishers/google/models/textembedding-gecko:predict',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': 'Bearer <Bearer token>'
      },
      data : data
    };

    axios(config)
    .then(function (response) {
      const documents = findSimilarDocuments(response.data.predictions[0].embeddings)
      console.log("Fetched docs")
      console.log(documents)
    })
    .catch(function (error) {
      console.log(error);
    });

}

async function findSimilarDocuments(embedding) {
    const url = 'MongoDB URI'; // Replace with your MongoDB url.
    const client = new MongoClient(url);

    try {
        await client.connect();

        const db = client.db('Database name'); // Replace with your database name.
        const coll = db.collection('Collection Name'); // Replace with your collection name.
        console.log("Inside MongoDB try")
        console.log(embedding["values"])

        // Query for similar documents.
        const documents = await coll.aggregate([{
          "$search": {
            "index": "default",
            "knnBeta": {
              "vector": embedding["values"],
              "path": "embedding_data",
              "k": 10
            }
          }
        },
        {"$project":{
            "_id": 0
        }}]).toArray();

        return documents;
    } finally {
        await client.close();
    }
}

```



