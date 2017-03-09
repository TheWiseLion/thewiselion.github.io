---
layout: post
title: "Lessons Learned In Migrating From Standard To Flex Environment (Python)"
categories: journal
tags: [documentation,sample]
image:
  feature: Port1024x600.png
  teaser: Port1024x380.png
---

# Background
Recently I had to change Google App Engine environments due to my project requiring a python extension, which aren’t typically supported in the “standard” environment (the exception is numpy). There are pretty vast difference between the environments, but the thing that made the flex enviroment most attractive to me is that the charge back model is based on exact cpu, memory, and disk usage instead of instance hours. This becomes relevant when you have really small applications that have little to no usage which is likely how many of my project will be positioned. A full comparison of the two environments can be seen here: https://cloud.google.com/appengine/docs/the-appengine-environments

---

# App.yaml
The first thing that has to be changed is the configuration file that determines how the app is deployed and in what enviroment. The allowed parameters are completely different based on the enviroment; in fact they don’t necessarily allow/require the same parameters. 

Compare the difference in the app.yaml files in both enviorments:
 
```yaml
# app.yaml (standard)
runtime: python27
api_version: 1
threadsafe: true

handlers:
- url: /.*
  script: main.app

#[START env]
env_variables:
    CLOUD_STORAGE_BUCKET: khet-online-storage
#[END env]
```


```yaml
# app.yaml (flex)
runtime: python
env: flex
entrypoint: gunicorn -b :$PORT main:app

runtime_config:
  python_version: 2

handlers:
- url: /.*
  script: main.app

#[START env]
env_variables:
    CLOUD_STORAGE_BUCKET: khet-online-storage
#[END env]
```

Probably the most notable is the "env" property is required in flex app.yaml but not valid in the standard environments configation, and yields vague 500 errors when trying to deploy your app.

---

# Storage
This is where the majority of the pain came from for me. I had originally used wonderful libraries like [ndb](https://cloud.google.com/appengine/docs/standard/python/ndb/) which functions much like ORMs like SQLAlchemy but for Google Cloud Storage; these however are not functional in the flex environment, which means I had to actually change significant amount of code…

**Something like this (standard env):**
```python
from google.appengine.ext import ndb

# Define Model
class Task(ndb.Model):
    """Models an individual Guestbook entry with content and date."""
    description = ndb.StringProperty()
	
# Create new entity
task = Task(id='simpletask1', description= 'Buy Milk')

# Insert new entity
task.put()
```
**Becomes like this (flex env):**
```python
# Imports the Google Cloud client library
from google.cloud import datastore

# Instantiates a client
datastore_client = datastore.Client()

# The kind for the new entity
kind = 'Task'
# The name/ID for the new entity
name = 'sampletask1'
# The Cloud Datastore key for the new entity
task_key = datastore_client.key(kind, name)

# Prepares the new entity
task = datastore.Entity(key=task_key)
task['description'] = 'Buy milk'

# Saves the entity
datastore_client.put(task)
```
As you can see the ndb library helps save quite a lot of boiler plate....

---

# Testing
One of the things I actually loved about the standard environment is that it was deployable on my local desktop ([dev_appserver.py](https://cloud.google.com/appengine/docs/standard/python/tools/using-local-server) tool), this allowed me to work out all the bugs before I ever needed to even try to deploy my app. Unfortunately the flex environment offers no equivent functionality, although we’re not totally left to the wolves either. The google cloud SDK offers a [datastore emulator](https://cloud.google.com/datastore/docs/tools/datastore-emulator) which was predominate feature I was utilizing in the local deployable server for the standard enviroment. It requires a bit more effort to setup as you'll like need to set specific environment variables that get are used by [datastore library](https://github.com/GoogleCloudPlatform/python-docs-samples/blob/master/datastore/cloud-client/quickstart.py).
```python
    os.environ['DATASTORE_DATASET'] ='YOUR DATASTORE'
    os.environ['DATASTORE_EMULATOR_HOST'] = 'localhost:PORT'
    os.environ['DATASTORE_EMULATOR_HOST_PATH'] = 'localhost:PORT/datastore'
    os.environ['DATASTORE_HOST'] = 'http://localhost:PORT'
    os.environ['DATASTORE_PROJECT_ID'] = 'YOUR PROJECT ID'
```
In conjunction with the emulator command
```bash
gcloud beta emulators datastore start --host-port=127.0.0.1:PORT --no-store-on-disk  --project=YOUR_PROJECT_ID
```

---

# Opinions
The google app engine space in my opinion is still in an early phase so you should expect to get burned when being an early adopter. While I still think the deployment process is better then what’s present at my corporate job, it certainly not especially easy as there are tons of potential pitfalls and non-obvious errors. 
<br/><br/>
My prediction is that the flex environment will see at least one backwards breaking change this year or perhaps be just be demised all together (see flex support policy), so stay tuned for the next post on how I had to fix my app in a few months from now…

![](https://cdn.scotch.io/23785/Ax1BxHPLQs6p23LML6FM_Demise.png "flex support/sla")
