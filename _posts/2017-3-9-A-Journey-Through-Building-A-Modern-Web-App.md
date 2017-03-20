---
layout: post
title: "Khet Online - An Unexpected Journey"
categories: [python, google_cloud, angularjs]
tags: [python, pypi, pip, google_cloud, angularjs]
image:
  feature: WebJourney.png
  teaser: WebJourney.png
sitemap:
  priority: 0.8
  changefreq: 'monthly'
---

# Synopsis
I chronicle my journey of building [khet-online](http://www.khet-online.com/) from selecting a web stack to deployment and support.
<br/><br/>

---

<br/>
# Background
Most of my projects stem from odd places and this one is no different. 
In this case my younger brothers got chess like board game called [Khet](https://en.wikipedia.org/wiki/Khet_(game)) and they bested me a few time at the game. 
Now getting beaten by a 10 year old at anything is ~~a bit~~ **extremely** embarrassing, 
so I felt I needed to bring all my knowledge to bear and make an AI that could eternally best them.
Hence I created [Khet-Online](http://khet-online.com/#!/about) and the corresponding PyPi library [pykhet](https://pypi.python.org/pypi/pykhet).
This post discusses some of the design choices a went through while developing the site.
<br/><br/>


---

<br/>
# Choosing a web stack
Python tends to my framework of choice for short term projects that I don't plan on maintaining far into the future. 
A lot of this has to do with the fact that in my experience python is a language that has immense library support and is great for rapid prototyping.
Perhaps this was an opportunity to branch out and try the ever growing technologies like [express.js](http://expressjs.com/) or [Play](https://www.playframework.com/), 
but I wanted to continue to explore the python space I was familiar with.
There is something to say about having dependable tooling and not wanting to spend the over head of learning a completely new languge with an entirely new tool chain.
<br/><br/>
In general I wanted to break things into three pieces: <br>
1. AI & Game Logic 
2. Restful web service that had corresponding [swagger docs](http://swagger.io/)
3. A web based UI (javascript/angularjs)

Based on this I ended up breaking the project into two repositories [Pykhet](https://github.com/TheWiseLion/pykhet) and [Khet Online](https://github.com/TheWiseLion/KhetOnline). 
It made sense to me that the game logic should be able to stand as its own project to be used and potentially contributed to by anyone that has similar set of requirements as me.

<p align="center">
    <img src="/images/ProjectLayouts.png"><br/>
	<em>Khet Online depends on Pykhet</em>
</p>

Most admirable web application take this basic form of heavy front end app (angularjs, react, vue, backbone) and a strong restful/backend framework (express, spring, django, play).
I've grown to accept this as the correct paradigm for most forms of [the single page web app](https://en.wikipedia.org/wiki/Single-page_application), which was my intent.
<br><br>
Python has a few options for strong web frameworks:
1. Django
2. Pyramid
3. Flask

I had used Django for other projects and it doesn't lend itself well as a minimalistic framework. It's a great if you plan on having a lot of [templatized](https://docs.djangoproject.com/en/1.7/topics/templates/#templates) static pages driven by some kind of database.
Instead I looked for a backend framework that emphasized serving rest services and generating swagger docs/specs. Swagger is a specification standard that provides a great interface for documentation/testing of rest services.

The best option I found for providing restful services with automatic generation of swagger spec was the minimalists [Flask framework](http://flask.pocoo.org/) with [Flask-RestPlus](https://flask-restplus.readthedocs.io/en/stable/) extension.
<br><br>
Consider the following snippet below that uses flask/flask-restplus. <br/>
With less than 50 lines we get the following rest service + swagger doc:
```python
from flask_restplus import Api, Resource, abort
from flask import Flask, Blueprint

# Define Rest API
rest_api = Api(version='1.0', title='modulo-api', description='Services For preforming modulo')
modulo_service = rest_api.namespace('modulo', description='Modulo Service')

# Define inputs to our service (get parameters)
get_input = modulo_service.parser()
get_input.add_argument('dividend', type=int, help='number to take modulo of (dividend)', required=True)
get_input.add_argument('divisor', type=int, help='the divisor', required=True)


@modulo_service.route('/modulo')
@modulo_service.response(400, "Bad Request Parameters")
class ModuloService(Resource):
    @modulo_service.expect(get_input)
    def get(self):
        """
        Modulo service takes dividend and divisor. 
		Divisor must not be zero.
        [dividend mod divisor]
        :return: number
        """
        args = get_input.parse_args()
        if args.divisor == 0:  # throw error
            abort(400, 'Divisor cannot be zero!')
        return args.dividend % args.divisor


# Define Flask Application
app = Flask(__name__)

# A bit of application setup boiler plate
app.config['SWAGGER_UI_DOC_EXPANSION'] = 'list'
app.config['RESTPLUS_VALIDATE'] = True
app.config['RESTPLUS_MASK_SWAGGER'] = False
app.config['ERROR_404_HELP'] = False

# Add swagger docs to localhost/api
blueprint = Blueprint('api', __name__, url_prefix='/api')
rest_api.init_app(blueprint)
app.register_blueprint(blueprint)

if __name__ == "__main__":
    # Start web service on port 80
    app.run(debug=True, port=80)
```

Which yields the beautiful swagger doc:
<p align="center">
    <img src="/images/swagger.png"><br/>
	<em>Rest Service + Generated Swagger Interface</em>
</p>

Of course this is a simplistic example but the setup for django is far more intricate and requires far more code to get a basic rest service running.
<br/><br/>

---

<br/>
# Building A Distributable Python Package
For me step one was building the board game logic in pure python. I knew I wanted to distribute my code through [PyPi](https://pypi.python.org/pypi) the primary package manager for python.
<br>
There is just something mystical about having your package/library available within one single command: 
```bash
pip install pykhet
```

My general goals were as follows:
1. Encapsulate all the logic of the game Khet
2. Support most versions of python (2.7 to 3.6)
3. Provide implicit support easy support for Json serialization
4. Provide enough documentation and examples that it might be used and extended by someone else

I'm not going to go into the details about my python library as it's just enormous amounts of condition statements and enums.
<br/><br/>
However one aspect that's worth discussing is [Travis CI](https://travis-ci.org) a build, test, and deploy tool. Travis CI provides hooks such that any time a commit is made against a git repository it will run your tests against as many versions of python as you want. 
With minimal setup you can get Travis CI to run your tests and across multiple versions of python every time there is a commit ([continuous integration](https://en.wikipedia.org/wiki/Continuous_integration)).
```yaml
# .travis.yaml - Travis Config File
language: python
python:
  - "2.7"
  - "3.2"
  - "3.3"
  - "3.4"
  - "3.5"
# command to install dependencies
install:
  - "pip install ."
# command to run tests
script: "nosetests"
```
This becomes really awesome because it lets me test multiple version of python support without really needing to do anything locally. 
At first my tests came back looking like this:
<p align="center">
    <img src="/images/FailedBuild.png"><br/>
	<em>Build + Tests Failing</em>
</p>

But by reading through errors generated by each version and making a few edits it ended up looking like this:
<p align="center">
    <img src="/images/PassBuild.png"><br/>
	<em>Build + Tests Passing</em>
</p>

You can also setup Travis CI to deploy code into things like google app engine or elastic bean stalk: <br/>
[https://docs.travis-ci.com/user/deployment](https://docs.travis-ci.com/user/deployment)
<br/><br/>
Another notable tidbit is how one gets their package/library on [PyPi](https://pypi.python.org/pypi).
It only requires two files (setup.py and setup.cfg) and an account on [PyPi](https://pypi.python.org/pypi) which is verified by a simple confirmation email. 
This to me is a bit surprising as there is so little security validation that I'm surprised [PyPi](https://pypi.python.org/pypi) isn't bloated with more code/packages.
Full guides on how to do this can be found [here](http://peterdowns.com/posts/first-time-with-pypi.html) and [here](https://www.codementor.io/arpitbhayani/host-your-python-package-using-github-on-pypi-du107t7ku).
<br/><br/>

---

<br/>
# Building The Site UI & Layout
The board game Khet is themed around lasers and Egyptian style board pieces. I felt a darker themed website would be the correct style choice. 
<br/><br/>
My initial (horrific) sketch was as follows:
<p align="center">
    <img src="/images/ConceptArt.png"><br/>
	<em>Initial Site Sketch</em>
</p>
<br/><br/>
While the final results were passible:
<p align="center">
    <img src="/images/FinalLayout.png"><br/>
	<em>Final Layout</em>
</p>
The choice for a dark themed website is one that should be made with careful consideration. 
Not only have darker websites been shown to be harder to read ([see here](http://ux.stackexchange.com/questions/53264/dark-or-white-color-theme-is-better-for-the-eyes)) most of the default themes provided by [material libraries](https://material.angularjs.org) are made for white or light backgrounds and a significant amount of effort had to be put in to make many elements suitable for a dark theme.
<br/><br/>

---

<br/>
# Building The Game UI
Unlike the most Khet apps I wanted to choose a strictly 2D representation as I felt that if it works for games like Chess so too should it work for Khet.
2D representations are easier to understand and in my opinion are better suited for representing board games.

I decided on using the library called createjs as it seems to specialize at 2D rasterization.
<br/><br/>
I hand crafted an image for each piece:
<p align="center">
    <img src="/images/first.png"><br/>
	<em>Initial Board</em>
</p>

The pieces weren't the same size nor consumed all the space of the square which creates quite a jarring effect.
<br/><br/>
After reworking the images:
<p align="center">
    <img src="/images/final.png"><br/>
	<em>Final Board With Laser Path</em>
</p>

It looks quite a bit better when reworked. I had to rely on distinct colors and mouse over events so that it would become self evident on how to play game as I didn't want instructions to be necessary.
<p align="center">
    <img src="/images/Move.png"><br/>
	<em>Piece selected (orange tile). Green tiles represent available moves.</em>
</p>

<br/><br/>

---

<br/>
# Building The Rest Services
I built the rest services using the minimalist [Flask framework](http://flask.pocoo.org/) with the [Flask-RestPlus](https://flask-restplus.readthedocs.io/en/stable/) extension.
The final rest services can be found at [this swagger page](http://khet-online.com/api/) it's mostly composed of apis to create, search, join, and play games.
<br/><br/>
However when building an interactive game rest services may not actually be the best approach as the game state has to constantly poll for new updates instead of getting notified when an update occurs. 
Instead something like [websockets](http://stackoverflow.com/questions/29145970/websocket-rest-client-connections) should be considered as they are more efficient and all around better when managing a stateful game.
<br/><br/>
A full write up on when one is more appropriate can be found [here](https://www.pubnub.com/blog/2015-01-05-websockets-vs-rest-api-understanding-the-difference/).
<br/><br/>

---

# Deploying The App
I had used AWS's [Elastic Beanstalk](https://aws.amazon.com/elasticbeanstalk/) in the past and I wanted to try Google's competing product [Google App Engine](https://cloud.google.com/appengine/).
<br/><br/>
It was pretty surprising how unpolished the deployment process was, it ended up taking quite a bit of effort to get all the dependencies working. You can see a full write up on this process [on my article](todo.com) about it.
When I added a python extension to pykhet module I had to redeploy using a different App Engine environment (flex environment) which again took more effort then I had anticipated. The my write up on the process can also be found [here](todo.com). 
<br/><br/>
Despite all my complaining I really think Google App Engine is a great service. The deployment and scaling process is light years ahead of the in house process I use at my day job. 
The entire concept of needing to know what machines my software is deployed on is completely removed which in my opinion is the right direction.
<br/><br/>

---

<br/>
# Support And Monitoring
Monitoring is mostly driven from logs events. 
<p align="center">
    <img src="/images/StackDriver.png"><br/>
	<em>Google's Log Service - StackDriver</em>
</p>
Specifically GAE allows you to watch specific criteria like certain rest calls or just a general responses like 404. This can get as granular or as fine as you want.
In my case I wanted to get an alert any time someone created a new game.
<br/><br/>
First you define a metric:
<p align="center">
    <img src="/images/CreateMetric.png"><br/>
	<em>Creating Metric</em>
</p>
<br/>
Then you can create an alert based on a metric using [strackdiver](https://www.stackdriver.com/):
<p align="center">
    <img src="/images/AlertingOnThreshold.png"><br/>
	<em>Creating Stackdriver Alert</em>
</p>



I used [Google's domain name registration service](https://domains.google/#/), which is a bit more costly then the ones I'm use to like [NameCheap](https://www.namecheap.com) and [GoDaddy](https://www.godaddy.com/). 
It has a really minimalist approach which I ended up liking since I really only needed a couple of basic features like email forwarding
<p align="center">
    <img src="/images/support.png"><br/>
	<em>Email Forwarding For Support</em>
</p>
<br/><br/>

---

<br/>
# Future Additions
Currently the AI is pretty bad and is still quite beatable. I'm planning on using [tensorflow](https://www.tensorflow.org/) to create a [deep Q network (DQN)](https://www.cs.toronto.edu/~vmnih/docs/dqn.pdf) that hopefully truly thrashes me at the game.