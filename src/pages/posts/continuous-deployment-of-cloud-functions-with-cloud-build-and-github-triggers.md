---
description: Continuous deployment of Cloud Functions with Cloud Build and GitHub Triggers
slug: continuous-deployment-of-cloud-functions-with-cloud-build-and-github-triggers
title: Continuous deployment of Cloud Functions with Cloud Build and GitHub Triggers
createdAt: 1694693347808
updatedAt: 1695644821880
tags:
  - Google Cloud
  - Cloud Functions
  - Cloud Build
  - Python
  - Github
heroImage: >-
  /posts/continuous-deployment-of-cloud-functions-with-cloud-build-and-github-triggers_thumbnail.png
public: true
layout: ../../layouts/BlogPost.astro
---


## Overview

In this article, i will show you how i automated the process of working with Cloud Functions in Google Cloud by creating a simple pipeline using Cloud Functions, Cloud Build and GitHub Triggers. If you use Cloud Functions on a daily basis and you want to simplify maintenance, this solution will help you save a lot of time. 

By linking your GitHub account to Cloud Build Repositories and setting up build triggers based on GitHub triggers you can listen to changes made in the GitHub repository and use Cloud Build to build all Cloud Functions. By doing so you eliminate the need to go in manually and change any Cloud Function settings/code in the Google Cloud interface.

Once you've completed the setup you will be able to make edits to your local repository and once you push changes to the main branch the updates will be automatically built with Cloud Build and update your Cloud Functions.

```
project
│   .env
│   .gitignore
│   .cloudbuild.yaml
│   main.py
│   README.md
│   requirements.txt
│
└───bin
│   │   setup.sh
│   │   functions.sh
│   
└───functions
│   │
│   └───function_one
│       │   main.py
│   │
│   └───function_two
│       │   main.py
│   │
│   └───function_three
│       │   main.py
```

## Prerequisites

- You need to create a Gmail or Google Workspace account
  - https://accounts.google.com/signup
- You need to be signed into Google Cloud
  - https://cloud.google.com/
- You need to enable billing for your Google Cloud account
  - https://console.cloud.google.com/billing
- Install the gcloud CLI on your local computer
  - https://cloud.google.com/sdk/docs/install-sdk
- Install git using one the following options
  - https://git-scm.com/download/

## Setup

### Fork the GitHub repository

Fork this repository and make a clone of the fork to your local computer
- https://github.com/hampussanden/cloud_functions_cloud_build_github_triggers

Here are the steps explained on how you do this
- https://docs.github.com/en/get-started/quickstart/fork-a-repo

### Set your environment variables

Create an .env file in your root directory and set your environment variables.

```shell
PROJECT_ID="your-project-id" # The project id you want your Google Cloud project to have
PROJECT_NAME="your-project-name" # The project name you want your Google Cloud project to have
REGION="your-project-region" # The region you wish to deploy this project in, https://cloud.google.com/about/locations
BILLING_ACCOUNT_ID="your-billing-account-id" # The billing account id that was generated when you setup up billing for your account 
CONNECTION_NAME="your-connection-name" # The connection name for the Google Cloud/GitHub connection eg. your github account name
REPO_NAME=cloud_functions_cloud_build_github_triggers # The name of your repository
REPO_URI=https://github.com/YOUR_USERNAME/cloud_functions_cloud_build_github_triggers.git # The URI of your repository
BRANCH_PATTERN=.* # The branch pattern that the build triggers listens to
BUILD_CONFIG_FILE=cloudbuild.yaml # The Cloud Build config file
MEMORY="your-cloud-function-memory" # The memory allocation for your Cloud Function, https://cloud.google.com/functions/docs/configuring/memory
```

### Run setup.sh

The shell script I've created to automate this process does the following:

- Creates a project
- Enables billing for the project
- Enables all the necessary APIs 
- Sets all the necessary permissions
- Creates a GitHub account connection
- Creates a GitHub repository connection
- Reads the functions folder and creates build triggers for your Cloud Functions

Make sure you have completed all the steps above before you run the setup.sh script.

```
. ./bin/setup.sh
```

### Adding a new function

The project currently contains three example functions. If you would like to add more functions you just follow these simple steps.

- Go to your /functions folder and create a new folder with your function name e.g. function_four

```
└───functions
│   │
│   └───function_one
│       │   main.py
│   │
│   └───function_two
│       │   main.py
│   │
│   └───function_three
│       │   main.py
│   │
│   └───function_four
│       │   main.py
```

- Add a main.py file inside that folder and write your function

```python
def main(event, context):
    """Background Cloud Function to be triggered by Pub/Sub.
    Args:
         event (dict):  The dictionary with data specific to this type of
                        event. The `@type` field maps to
                         `type.googleapis.com/google.pubsub.v1.PubsubMessage`.
                        The `data` field maps to the PubsubMessage data
                        in a base64-encoded string. The `attributes` field maps
                        to the PubsubMessage attributes if any is present.
         context (google.cloud.functions.Context): Metadata of triggering event
                        including `event_id` which maps to the PubsubMessage
                        messageId, `timestamp` which maps to the PubsubMessage
                        publishTime, `event_type` which maps to
                        `google.pubsub.topic.publish`, and `resource` which is
                        a dictionary that describes the service API endpoint
                        pubsub.googleapis.com, the triggering topic's name, and
                        the triggering event type
                        `type.googleapis.com/google.pubsub.v1.PubsubMessage`.
    Returns:
        None. The output is written to Cloud Logging.
    """
    import base64

    print(
        """This Function was triggered by messageId {} published at {} to {}
    """.format(
            context.event_id, context.timestamp, context.resource["name"]
        )
    )

    if "data" in event:
        name = base64.b64decode(event["data"]).decode("utf-8")
    else:
        name = "World"
    print(f"Hello {name}!")
```

- Go to the main.py file in your root directory and add your new function

```python
from functions.function_one import main as function_one
from functions.function_two import main as function_two
from functions.function_three import main as function_three
from functions.function_four import main as function_four

def function_one_entry(event, context):
    function_one.main(event, context)

def function_two_entry(event, context):
    function_two.main(event, context)

def function_three_entry(event, context):
    function_three.main(event, context)
    
def function_four_entry(event, context):
    function_four.main(event, context)
```    

## FAQ

### How much does this setup cost?

This project uses Cloud Build and Cloud Functions which would not incur a lot of cost to your account. Here is the [Google Cloud Functions pricing](https://cloud.google.com/functions/pricing) for calculation.

### How many Cloud Functions can i create?
You can create as many functions as you like as long as you don't reach any limits of usage in Google Cloud.

### Will deleted Cloud Functions be removed automatically?
Not in this version but I'm planning on adding it in the future.

### Do i need to run the script when I've created a new Cloud Function in the functions folder?
Yes, you will need to run the setup.sh script again.

### How can I update the Cloud Functions?
Once you have made this setup the function will update on each push you make to your GitHub repository.

### What type of Cloud Functions are used in this setup?
The Cloud Functions i use in this example are 1st gen and uses pub/sub triggers.

### Are there any known security implications with this setup?

You can read more about the security implications of using build triggers here:

- https://cloud.google.com/build/docs/automating-builds/create-manage-triggers#security_implications_of_build_triggers