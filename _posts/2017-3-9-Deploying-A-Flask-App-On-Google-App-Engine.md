---
layout: post
title: "Deploying A Flask App On Google App Engine's Standard Environment"
categories: journal
tags: [documentation,sample]
image:
  feature: DOGAE.png
  teaser: DOGAE.png
---

# Background
I couldn’t find an easy guide to follow other than the [google app engine's lengthy tutorial](https://cloud.google.com/appengine/docs/python/getting-started/creating-guestbook), so in hope of saving others a bit of time I made this nifty post. In this post I go over deploying a flask application in google app engines ‘standard’ environment and some potential pitfalls you might encounter on the way. 

-----
# Google App Engine Environments
Google app engine (GAE) offers two possible enviroments, [standard and flex](https://cloud.google.com/appengine/docs/the-appengine-environments). The two enviroments are mostly the same however the flex enviroment has better library support and pricing model; note however the flex environment is still in beta and has significantly less tooling. Currently the google app engine's provided local dev server ([dev_appserver.py](https://cloud.google.com/appengine/docs/standard/python/tools/using-local-server))  only supports the standard environment. Based on this I decided to release on the standard environment as I felt that it’d be faster to reach a working deployment if I could fail very quickly.

-----
# Step 1: Create an App ID
You'll first need to create an application on google cloud services which you can get to by going to the [main console](https://console.cloud.google.com/).
![](https://cdn.scotch.io/23785/d9IqCuxBREOkHnWceMBH_create%20project.png "Create Project") 

-----
# Step 2: Enabling Services
You may need to enable various services before deploying. If you navigate to the consoles of the various services it should ask if you want the service enabled for your application
I enabled:
* [App Engine](https://console.cloud.google.com/appengine)
* [Compute Engine](https://console.cloud.google.com/compute)
* [Storage](https://console.cloud.google.com/storage)

-----
# Step 3: Setting up your local environment
For this next sections you probably want to make sure your using a virtual environment and produce the correct requirements.txt file. The final results will yield the relatively few files you need to add to work with google app engine. You'll also need to be sure you have the [SDK for App Engine](https://cloud.google.com/appengine/docs/python/download)
![](https://cdn.scotch.io/23785/Br3SxyLRr6KC0CDPwtxl_project%20layout.png "Required Files For GAE")
<br/>
**Creating Configuration File (app.yaml):**<br/>
The app.yaml file describes properties on how to handle routing and the deployment your application.
```yaml
# app.yaml
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

The runtime argument is always python27 for standard environment. 
The handlers are used for routing to different components (e.g. [static content](https://cloud.google.com/appengine/docs/python/getting-started/serving-static-files)). 
Specifically for my site everything is routed to my main.py:
```python
import ...
# Google App Engine will look for this variable (".app" in handler)
app = Flask(__name__)
```

If your using cloud storage in your app you can specify the "CLOUD_STORAGE_BUCKET" environment variable and that's what libraries the [ndb](https://cloud.google.com/appengine/docs/python/ndb/) will use as the default bucket. 

**NOTE** <br/>
When I was making my app.yaml assumed the allowed properties were the same regardless of which environment your deploying into. <br/>
This caused the extremely nondescript error: <br/>
```bash
	Updating service [default]...failed. 
	ERROR: (gcloud.app.deploy) INVALID_ARGUMENT: The request is invalid.
```
Many wasted hours because the key "env" is required when deploying on flexible environment, but not allowed when deploying on the standard enviroment. Below is an example of the app.yaml for the flex enviroment.

```yaml
# app.yaml for flex enviroment
runtime: python
env: flex
entrypoint: gunicorn -b :$PORT main:app

runtime_config:
  python_version: 2

handlers:
- url: /.*
  script: main.app
```


**Adding Libraries And Dependencies:** <br/>

Unlike the flex environment the standard environment won't use your requirements.txt 
Instead you must bundle the required libraries in a folder. When you create the appengine_config.py file in your root folder you can configure additional locations for where to find requisite libraries.
<br/><br/>
I used "**pip install -t lib -r requirements.txt**" to save all the required libraries to a deployable folder 
 Further instructions can be found [here](https://cloud.google.com/appengine/docs/python/tools/using-libraries-python-27#installing_a_third-party_library)
<br/><br/>
Technically you can[ add specific libraries in your app.yaml](https://cloud.google.com/appengine/docs/python/tools/built-in-libraries-27) but I was already so skeptical that I figured I avoid the frustration by just loading all my required files with the mechanism I knew to work.

-----

# Step 4: Local Testing
**Setting up gcloud:**

If this is the first app your setting up you should run: <br/>
`gcloud init` <br/>
You'll get a prompted to set default regions, email address, and project id.


You should also set the default project with: <br/>
'gcloud config set project `<Your Project ID>` <br/>

You should also login with: <br/>
`gcloud auth application-default login` <br/>

I had some weird threading bug on my client and the recommended fix was running:
gcloud config set app/num_file_upload_processes 1
Once that's set up you can run your app locally using 'dev_appserver.py : <br/>
`dev_appserver.py --port <port num> <path\to\your\app.yaml>`<br/>


** NOTE ** <br/>
I had trouble with all flask versions greater than 0.10.2 as the dependent library "click" fails to work with the local server, so I decided I could live with a lower version of Flask.....

-----

# Step 5: Deploying 
Once you've fixed all the kinks in your local environment you can deploy by using: <br/>
`gcloud app deploy app.yaml`
I had problems with some of my files not getting deployed so you may need to change the [standard skipped files](https://cloud.google.com/appengine/docs/python/config/appref#skip_files).

![](https://cdn.scotch.io/23785/YbjzXEiTyardmcasYbJq_console.png "What it should look like if things work.....") 


**NOTE** <br/>
If you end up with the message "**INVALID_ARGUMENT: The request is invalid.**" validate your app.yaml is correct, as there is less stringent validation of this file when testing using the local server.

-----

# Going Past Deployment
Originally google actually offered a service to deploy directly from your repository but they deprecated this feature. I find this particularly sad because that's one of my favorite features offered by competitors like elastic bean stalk. It's especially aggregating as it wasn't always clear on app engine what files were going actually to get deployed, as by default app engine skips various files (e.g. those beginning with "." or ending with ".pyc")

An alternative is to use Travis CI's to trigger an app cloud update after a success build. 
https://docs.travis-ci.com/user/deployment/google-app-engine/