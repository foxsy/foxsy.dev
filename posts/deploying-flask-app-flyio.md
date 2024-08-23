---
title: 'Deploying a Flask App to Fly.io'
pubDate: 2024-08-25
description: 'Deploy a Flask App to fly.io'
author: 'Foxsy'
image:
    url: 'https://foxsy.dev/blog-images/images/python-flask-fly-io.webp'
    alt: 'Python flask logo with colorful wings.'
tags: ["fly.io", "flask", "python", "deploy"]
heroImage: 'https://foxsy.dev/blog-images/images/python-flask-fly-io.webp'
---

Today's post will be about deploying a very basic Flask app to [fly.io](https://fly.io).  Usually when I build an backend
API, I tend to stick with the "what I already know" frame of mind.  The problem is that sometimes I want to build something in a specific language because
it has better libraries or capabilites for my particular need.  In my case the "what I already know" usually means [Cloudflare Workers](https://workers.cloudflare.com/) as this
makes it easy to build something without worrying about scaling or cost (though cost can become an issue if utilization starts to ramp up).  One issue I've run into recently is
the limitations of [Cloudflare Workers](https://developers.cloudflare.com/workers/platform/limits/).  While Cloudflare Workers does support JavaScript and
TypeScript (and it does list Python and Rust in the [supported languages](https://developers.cloudflare.com/workers/languages/)).  There are limits to what libraries you can use
in these languages.  For instance in JavaScript and TypeScript you can't use most `npm packages` or `http libraries`.  The pattern I use, when using JavaScript or TypeScript, seems
to be using the `itty-router-openapi` which has been renamed to [chanfana](https://github.com/cloudflare/chanfana).  I have tried using their new support for [Python Workers](https://developers.cloudflare.com/workers/languages/python/packages/)
but it falls into the same category of having a lot of limitations, mostly only supporting [FastAPI](https://developers.cloudflare.com/workers/languages/python/packages/fastapi/) and some other
base Python pacakges that are listed [here](https://developers.cloudflare.com/workers/languages/python/packages/#supported-packages) and the ability to request support more packages.
There is a notice at the top of the page for [Python Workers](https://developers.cloudflare.com/workers/languages/python/packages/#package-versioning) that states they will eventually support specifying package in the `requirements.txt` and that it is currently `beta` and
cannot be deployed at this time. Until then, I needed an alternative to deploy my Flask apps that support any libraries that I needed.

This brings us to [fly.io](https://fly.io), which reminds me a little of [PikaPods](https://www.pikapods.com/) but allows you to deploy your own custom applications.
With [fly.io](https://fly.io) you can easily take your custom built application and deploy it via their CLI with a
[Dockerfile](https://fly.io/docs/languages-and-frameworks/dockerfile/).  This means that you can test your application locally using [Docker](https://www.docker.com/products/docker-desktop/)
and know that it will behave the same when deployed to [fly.io](https://fly.io).  You can also choose to use any of their
[language frameworks](https://fly.io/docs/languages-and-frameworks/) that they support, but for this use case we will use their [Flask App Guide](https://fly.io/docs/python/frameworks/flask/).

## Table of contents
- [Why I chose fly.io for hosting a Flask app](#whyflyio)
- [What You Will Need](#whatyouwillneed)
- [Setting Up Your Project](#setupyourproject)
- [Create Your Flask App](#createflaskapp)
- [Create a Dockerfile](#createdockerfile)
- [Deploying to Fly.io](#deploytoflyio)
- [Deploying to Fly.io With CI/CD](#deployflyiocicd)
- [What's Next?](#whatsnext)

<a id="whyflyio"></a>
## Why I chose fly.io for hosting a Flask app
While on a quest to find an inexpensive way to host a backend API that was written in [Go](https://go.dev/), I came across
Reddit threads where people were talking about [fly.io](https://fly.io) to host their site and backend.  I decided to give it a try
and I have to say that I'm pretty impressed with [fly.io](https://fly.io) so far.  Coming from a DevOps/SRE
background, it's interesting to see such a service exist.  Not to mention their [documentation](https://fly.io/docs/) and transparency in their
[architecture](https://fly.io/docs/reference/architecture/) made it feel less like some `secret sauce` that could be brittle behind the scenes.
The final piece that made it appealing was the cost/value versus hosting it in something like
[AWS](https://aws.amazon.com/?nc2=h_lg) or [GCP](https://cloud.google.com/).  The cost/transparency coupled with
[accident forgiveness](https://fly.io/blog/accident-forgiveness/) made this an easy choice for me.

<a id="whatyouwillneed"></a>
## What You Will Need
 - [ ] [GitHub](https://github.com) or [GitLab](https://gitlab.com) **(optional)**
 - [ ] [Docker Desktop](https://www.docker.com/products/docker-desktop/)
 - [ ] [Fly.io](https://fly.io/app/sign-up)
 - [ ] [Python and Flask](https://flask.palletsprojects.com/en/3.0.x/installation/)
 - [ ] [Fly.io CLI](https://fly.io/docs/flyctl/install/)

<a id="setupyourproject"></a>
## Setting Up Your Project
Once you have all of the basic requirements installed, you can start creating your Flask app that will be deployed to [fly.io](https://fly.io)

* Create working director and virtual environment for Python
 ```
mkdir flying-flask-app
cd flying-flask-app
python3 -m venv ./.venv
```
> **_NOTE:_** I typically put my `venv` within my working directory as this makes it easier for me to manage for a
specific project

* Create requirements.txt file so that we can add libraries we will need
```
touch requirements.txt
```

* Edit the requirements.txt and add the libraries we need
```
Flask==2.2.2
```

* Activate the Python virtual environment and install the requirements
```
source .venv/bin/activate
pip3 install -r requirements.txt
```

<a id="createflaskapp"></a>
## Create Our Flask App
* Create our main entrypoint for our Flask app
```
touch app.py
```

* Edit the app.py and we can add some initial code to test that things are working correctly
```
from flask import Flask

app = Flask(__name__)

@app.route("/")
def hello():
  return "Hello, World!"
```

Great! Now we can run our app to make sure it works!
```
python3 -m flask run --host=127.0.0.1 --port=8080
```
![Screenshot of flask app running on localhost](https://foxsy.dev/blog-images/images/flask-app-running-localhost.png)

You can see that it is now running on `localhost` port `8080`, so let's go check out our fancy new page!
![Screenshot of Flask app in a browser](https://foxsy.dev/blog-images/images/flask-app-running-localhost-browser.png)

I'll admit, this isn't very impressive... 😞 But, we can go ahead and add a few things before we deploy to
[fly.io](https://fly.io) to make this a little more exciting. Let's add a **GET** endpoint that processes a parameter
that is passed via url since this is somewhat common. Another thing we can add is parsing some `json` in a **POST**
request and formatting the output to display values from that `json`.  Lastly, we will add a `PATCH` endpoint that will call a third-party API to get some
data, parse it, and return our own json structure that includes that data.

* Processing a parameter value from the URL
```
from flask import Flask, request

app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello, World!"

@app.route("/sayhello")
def sayhello():
    args_name = request.args.get('name')
    hello_string = ""
    if not args_name:
        hello_string = "Howdy, stranger!"
    else:
        hello_string = f'Howdy {args_name}!'
    return hello_string
```
> **_NOTE:_** Be sure to add `request` as part of your import

* Restart the Flask app to test your changes
<a id="restartflask"></a>
**_CTRL+C_** to stop your current Flask app, then run the following or **_UP-ARROW ENTER_** to run the previous command
```
python3 -m flask run --host=127.0.0.1 --port=8080
```
Now when we visit [http://127.0.0.1:8080/sayhello](http://127.0.0.1:8080/sayhello?name=foxsy) with a name parameter
it will output the name we provided `http://127.0.0.1:8080/sayhello?name=Bob`
![Screenshot of Flask app with URL name parameter](https://foxsy.dev/blog-images/images/flask-app-running-localhost-name-param.png)

Great! 🥳 Now we can move on to a `POST` request!

* Adding a `POST` request with a `json` body
```
@app.route("/submit-contact", methods=['POST'])
def submit_contact():
    # Get the json body and store it in the data var
    data = request.get_json()

    # If the body is empty, return a 400 with error
    if not data:
        return jsonify({"error": "Invalid or missing JSON data"}), 400

    # Get the individual fields in the JSON passed
    name = data.get('name')
    email = data.get('email')

    # If either are empty, it's not valid for parsing so return a 400 with error
    if not name or not email:
            return jsonify({"error": "Missing required fields"}), 400

    # We made it this far, so this is where we would process and store the data to a database or do some
    # action with a third-party API, since that's not the goal now, we will just return a 200 with some details
    return jsonify({"message": "Contact submitted successfully", "name": name, "email": email}), 200
```
> **_NOTE:_** Be sure to add `jsonify` as part of your import statement at the top

This is a really simple example of parsing a `json` body and giving proper status code returns.  I commented as
much as I could so it was clear what each part does. 😃

Now to [restart flask](#restartflask) and check out our changes. I'm going to use Postman for testing this endpoint
since it makes it easier to visualize.  Feel free to use `curl` or any other tools to test sending `json` to this endpoint.

**If you want to use curl:**
```
curl --location 'http://localhost:8080/submit-contact' \
--header 'Content-Type: application/json' \
--data-raw '{
    "name": "John Doe",
    "email": "johndoe@example.com"
}'
```
![Screenshot of Postman sending json to Flask app](https://foxsy.dev/blog-images/images/flask-app-running-localhost-postman-submit-contact.png)

The final part of this Flask app will be an endpoint that uses the `PATCH` method to call another API and mesh some data together, then
return that data as `json`.

* Adding a `PATCH` endpoint that calls another API
You will need to add the `requests` library and import it into your `app.py`.  Edit your `requirements.txt` and add the library with version
number and it should look like this when you are done.
```
Flask==2.2.2
Werkzeug==2.2.2
requests==2.32.3
```

You will need to run the `pip install -r requirements.txt` again to install the added library, and `import` requests at the top of
your `app.py`


```
@app.route("/combined-data", methods=['PATCH'])
def combined_data():
    # Set the endpoint for the third-party api
    api_url = "https://jsonplaceholder.org/users/1"
    data = {
        "name": "John Doe",
        "email": "johndoe@example.com"
    }

    # Call the API in a try/except block to handle errors
    try:
        response = requests.get(api_url)

        # Check if the request was successful
        response.raise_for_status()

        # Parse the JSON response from the external API
        external_data = response.json()

        # Get the birthDate field in the response
        birth_date = external_data.get('birthDate')

        # Add the birth date field to our user dict
        data["birth_date"] = birth_date

        # Return the dict as json
        return jsonify(data), 200

    except requests.exceptions.HTTPError as http_err:
        # Handle HTTP errors (4xx and 5xx responses)
        return jsonify({"error": f"HTTP error occurred: {http_err}"}), response.status_code

    except requests.exceptions.RequestException as err:
        # Handle all other types of errors
        return jsonify({"error": f"An error occurred: {err}"}), 500
```

Now we can do the same steps as before to [restart flask](#restartflask) and test our endpoint. If you want to use `curl` you can run the following command.
```
curl --location --request PATCH 'http://localhost:8080/combined-data' \
--header 'Content-Type: application/json' \
--data-raw '{
    "name": "John Doe",
    "email": "johndoe@example.com"
}'
```
![Screenshot of curl returning response from flask app PATCH endpoint](https://foxsy.dev/blog-images/images/flask-app-running-localhost-curl-combined-content.png)

Yay! Now we are done! Well, except the part where we still need to deploy to [fly.io](https://fly.io).  Since we are going to use the pattern
of creating our own Dockerfile to deploy, let's go ahead and do that now!

<a id="createdockerfile"></a>
## Create a Dockerfile
* Create a new file called `Dockerfile` in your Flask app director
```
touch Dockerfile
```

Now you can open it in your favorite editor and add the contents needed to deploy
```
# syntax=docker/dockerfile:1

ARG PYTHON_VERSION=3.12.4

FROM python:${PYTHON_VERSION}-slim

LABEL fly_launch_runtime="flask"

WORKDIR /code

COPY requirements.txt requirements.txt

RUN pip3 install -r requirements.txt

COPY . .

EXPOSE 8080

# Bind to both ipv4 and ipv6 address
ENV GUNICORN_CMD_ARGS="--bind=[::]:8080 --workers=2"

# replace APP_NAME with module name
CMD ["gunicorn", "wsgi:app"]
```

All of the Dockerfile probably makes sense up to the last two lines about `gunicorn`.  We could just run this
in docker with the same command that we have been using locally `python3 -m flask run --host=127.0.0.1 --port=8080` but
there are a lot of concerns running it like this in `production`.  Performance and concurrency in `gunicorn` will be
significantly better than running directly with `python/python3`.  The second, and probably most important, is security.
`Gunicorn` is better equipped to handle a DoS attack or other known flaws in the built-in Flask server.

This means that you will need to add the following to your `requirements.txt` and run `pip install -r requirements.txt` again.
```
gunicorn==20.1.0
```

After doing this, you can actually run it locally with `gunicorn` if you wanted to test it
```
gunicorn --bind=[::]:8080 --workers=2 wsgi:app
```

Excellent! We are now ready to move on to the deployment part!

<a id="deploytoflyio"></a>
## Deploying to fly.io
* Create the `fly.toml` that describes your app and how it will be deployed to [fly.io](https://fly.io)
```
flyctl launch
```

This will scan your current directory and create a base `fly.toml` file.  It will then build the Docker image and deploy it
to the region and number of machines specified.  It will allow you to modify the initial settings at this step, but I
said no as I was fine with the default settings. I did say **yes** to the *.dockerignore* and *.gitignore* files

![Screenshot of flyctl creating initial fly.toml for app deployment](https://foxsy.dev/blog-images/images/flask-app-flyctl-toml-creation.png)

> **_NOTE:_** I had to specify the org with `--org` since I have multiple orgs in my account

When the build and deployment is completed it will output a DNS endpoint for you to access, it always runs on port `80/443`
and redirects to `443` for `SSL`.  This means we have deployed an API that is secure and has a DNS endpoint that's *easy* to remember.
![Screenshot of Flask app building and deploying on fly.io](https://foxsy.dev/blog-images/images/flask-app-deploying-flyio.png)

If we access the endpoint we can see that it deployed our Flask app.
![Screenshot of Flask app running on fly.io](https://foxsy.dev/blog-images/images/flask-app-running-flyio.png)

<a id="deployflyiocicd"></a>
## Deploying to fly.io with CI/CD
The really nice thing about using the `flyctl` command after we have set everything up is that it automatically creates a
`fly-deploy.yml` in your project folder if it's a `GitHub` repo.  This is great because that means all we have to do is set up our
token in `GitHub`.  You can create a token through the dashboard or using the `fly` command line tool

```
fly tokens create deploy
```

This will output a token that you can use in `GitHub` actions. Follow the `GitHub` instructions for
[adding secrets to a repo](https://docs.github.com/en/codespaces/managing-codespaces-for-your-organization/managing-development-environment-secrets-for-your-repository-or-organization#adding-secrets-for-a-repository) and
then you should be able to deploy your Flask app when you commit code to your repo.

## Checkout the repo for this project [here](https://github.com/foxsy/flying-flask-app)

<a id="whatsnext"></a>
## What's Next?
Although this was a very simple demonstartion of how you would deploy a Flask app to [fly.io](https://fly.io), you can see how this
allows you to have more verastility when building your projects.  If we've learned anything from this post, it's that often times
we have to overcome the limitations of what is normal for us, or adapt to remove those limitations entirely.  In a future post I will talk about deploying
a more complex Flask app that connects to [Convex](https://www.convex.dev) to read and store data and will be deployed to [fly.io](https://fly.io).
