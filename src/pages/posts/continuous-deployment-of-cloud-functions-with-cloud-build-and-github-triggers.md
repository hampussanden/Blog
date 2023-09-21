---
description: Continuous deployment of Cloud Functions with Cloud Build and GitHub Triggers
slug: continuous-deployment-of-cloud-functions-with-cloud-build-and-github-triggers
public: true
layout: ../../layouts/BlogPost.astro
title: Continuous deployment of Cloud Functions with Cloud Build and GitHub Triggers
createdAt: 1694693347808
updatedAt: 1695284023586
tags:
  - Google Cloud
  - Cloud Functions
  - Cloud Build
  - Python
  - Github
heroImage: >-
  /posts/continuous-deployment-of-cloud-functions-with-cloud-build-and-github-triggers_thumbnail.png
---


## Overview

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

Fork [this](https://github.com/hampussanden/cloud_functions_cloud_build_github_triggers) repository and make a clone of your fork down to your local computer
- https://docs.github.com/en/get-started/quickstart/fork-a-repo

### Set your environment variables

Create an .env file in your root directory and set your environment variables.

```shell
PROJECT_ID="your-project-id" # The project id you want your Google Cloud project to have
PROJECT_NAME="your-project-name" # The project name you want your Google Cloud project to have
REGION=europe-west1 # The region you wish to deploy this project in
BILLING_ACCOUNT_ID="your-billing-account-id" # The billing account id that was generated when you setup up billing for your account 
CONNECTION_NAME=cloud-functions # The connection name for the Google Cloud/GitHub connection
REPO_NAME=cloud_functions_cloud_build_github_triggers # The name of your repo
REPO_URI=https://github.com/YOUR_USERNAME/cloud_functions_cloud_build_github_triggers.git # The URI of your repo
BRANCH_PATTERN=.* # The branch pattern that the build triggers listens to
BUILD_CONFIG_FILE=cloudbuild.yaml # The Cloud Build config file
MEMORY="your-cloud-function-memory" # The memory allocation for your Cloud Function, https://cloud.google.com/functions/docs/configuring/memory
```

### Run setup.sh

```
. ./bin/setup.sh
```

## FAQ

### How much does it cost?

This project uses Cloud Build and Cloud Functions which would not incur a lot of cost to your account. Here is the [GCF pricing](https://cloud.google.com/functions/pricing) for calculation.

### Can i use only one Cloud Function?
You can use as many functions as you like as long as you don't reach any limits of usage in Google Cloud. To add a new function you just create a new folder with the function name and create a main.py file inside that folder then re-run the setup.sh script.

### How can I update a function?
Once you have made this setup the function will update on each push you make to your GitHub repository.

### Are there any known security implications with this setup?

You can read more about the security implications of using build triggers here:

- https://cloud.google.com/build/docs/automating-builds/create-manage-triggers#security_implications_of_build_triggers