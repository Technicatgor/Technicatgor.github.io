---
layout: post
title: Make a simple web app using Flask in Python
date: 2023-05-17 00:00:00 +800
categories: [Development,Web]
tags: [python,flask,venv]
---
![flask](https://upload.wikimedia.org/wikipedia/commons/thumb/3/3c/Flask_logo.svg/920px-Flask_logo.svg.png)

Flask is a small and lightweight Python web framework that provides useful tools and features that make creating web applications in Python easier. 

## Prerequisites
- Python3
- pip
- git (optional)

## Venv 
Starting python 3.3, module venv has been added to python standard library and can be used as a drop-in replacement for virtualenv.
```sh
mkdir flask_app && cd flask_app
python -m venv env
```
active venv
```sh
source env/bin/activate
```

you will see your environment is activated like this.
```sh
(env)heston@localhost:$
```
I use `git` that easy to control my repo version. Remember to add the `env` dir in `.gitignore`
```
git init

cat <<EOF >> .gitignore
env/
EOF
```
> **NOTE** - Go to [Link](https://gitignore.io) to check which thing should be ignore.

## Install Flask
```
pip install flask
```
check flask version
```
python -c "import flask; print(flask.__version__)"

2.3.2
```
Output installed packages in requirements format.
```
pip freeze > requirements.txt
```
cat requirements.txt
```
blinker==1.6.2
click==8.1.3
Flask==2.3.2
itsdangerous==2.1.2
Jinja2==3.1.2
MarkupSafe==2.1.2
Werkzeug==2.3.4
```
next time you can install the packages in easier way
```
pip install -r requirements.txt
```
## Create the app
vim app.py
```py
from flask import Flask

app = Flask(__name__)


@app.route('/')
@app.route('/index/')
def hello():
    return '<h1>Hello, World!</h1>'


@app.route('/about/')
def about():
    return '<h3>This is a Flask web application.</h3>'
```

```sh
export FLASK_APP=app.py
export FLASK_ENV=development
```

Run the app.
```sh
flask run --host=0.0.0.0 --port=5001
```
Go http://localhost:5001/about that will see\
This is a Flask web application.

## Dynamic Route
Use `escape()` function to render the word string as text. This is important to avoid XSS attacks, which can keep your web app safe.
```py
from markupsafe import escape

#...

@app.route('/capitalize/<word>/')
def capitalize(word):
    return '<h1>{}</h1>'.format(escape(word.capitalize()))
```

To respond with an HTTP 404 error, you will need Flask’s abort() function, which can be used to make HTTP error responses. Change the second line in the file to also import this function:
```py
from markupsafe import escape
from flask import Flask, abort

#...

@app.route('/users/<int:user_id>/')
def greet_user(user_id):
    users = ['Bob', 'Jane', 'Adam']
    try:
        return '<h2>Hi {}</h2>'.format(users[user_id])
    except IndexError:
        abort(404)
```

My final `app.py` file
```py
from markupsafe import escape
from flask import Flask, abort

app = Flask(__name__)


@app.route('/')
def hello():
    return '<h1>Hello, World!</h1>'


@app.route('/about/')
def about():
    return '<h3>This is a Flask web application.</h3>'


@app.route('/capitalize/<word>/')
def capitalize(word):
    return '<h1>{}</h1>'.format(escape(word.capitalize()))


@app.route('/message/<msg>/')
def get(msg):
    return {
            'status': 'SUCCESS',
            'message': msg
            }


@app.route('/users/<int:user_id>/')
def greet_user(user_id):
    users = ['Flask', 'Python', 'Django']
    try:
        return '<h2>Hi {}</h2>'.format(users[user_id])
    except IndexError:
        abort(404)
```
{: file="flask_app/app.py"}


