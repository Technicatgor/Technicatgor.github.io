---
layout: post
title: Logging Module for Python
date: 2024-05-30 13:00:00 +800
categories: [Dev]
tags: [python]
image:
  path: /assets/img/headers/python_cover.jpeg
  lqip: data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAA8gAAAH4CAIAAAAZ1VPRAALJIklEQVR4Aeyah5IbuxVE0aQ3h/eUv8L//1MOm/NumxaKp9TVZInO4e1d1QgDXNyMHgyG+vj59+Od
---

## Logging Levels

| Log Level | Numerical Value | Purpose                                                                                                                                   |
| --------- | --------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| DEBUG     | 10              | Provides detailed information for diagnosing code-related issues, such as printing variable values and function call traces.              |
| INFO      | 20              | Used to confirm that the program is working as expected, like displaying startup messages and progress indicators.                        |
| WARNING   | 30              | Indicates a potential problem that may not be critical to interrupt the program's execution but could cause issues later on.              |
| ERROR     | 40              | Represents an unexpected behavior of the code that impacts its functionality, such as exceptions, syntax errors, or out-of-memory errors. |
| CRITICAL  | 50              | Denotes a severe error that can lead to the termination of the program, like system crashes or fatal errors.                              |

### 1. Setting the log level

```py
import logging

# Create a logger
logger = logging.getLogger(__name__)

# Set logger level to DEBUG
logger.setLevel(logging.DEBUG)
```

### 2. Creating a Formatter

```py
formatter = logging.Formatter('%(asctime)s - %(levelname)s - %(message)s')
```

### 3. Creating Handlers

As discussed previously, handlers manage where your log messages will be sent. We will create two handlers: a console handler to log messages to the console and a file handler to write log messages to a file named 'app.log'.

```py
console_handler = logging.StreamHandler()
console_handler.setLevel(logging.INFO)
console_handler.setFormatter(formatter)

file_handler = logging.FileHandler('app.log')
file_handler.setLevel(logging.DEBUG)
file_handler.setFormatter(formatter)
```

Both handlers are then added to the logger using the `addHandler()` method.

```py
logger.addHandler(console_handler)
logger.addHandler(file_handler)
```

### 4. Testing the Logging Setup

Now that our setup is complete, let's test if it's working correctly before moving to the real-life example. We can log some messages as follows:

```py
logger.debug('This is a debug message')
logger.info('This is an info message')
logger.warning('This is a warning message')
logger.error('This is an error message')
logger.critical('This is a critical message')
```

When you run this code, you should see the log messages printed to the console and written to a file named 'app.log', like this:

**Console**

```py
2024-05-18 11:51:44,187 - INFO - This is an info message
2024-05-18 11:51:44,187 - WARNING - This is a warning message
2024-05-18 11:51:44,187 - ERROR - This is an error message
2024-05-18 11:51:44,187 - CRITICAL - This is a critical message
```

**app.log**

```py
2024-05-18 11:51:44,187 - DEBUG - This is a debug message
2024-05-18 11:51:44,187 - INFO - This is an info message
2024-05-18 11:51:44,187 - WARNING - This is a warning message
2024-05-18 11:51:44,187 - ERROR - This is an error message
2024-05-18 11:51:44,187 - CRITICAL - This is a critical message
```

## Logging User Activity in a Web Application

In this simple example, we will create a basic web application that logs user activity using Python's logging module. This application will have two endpoints: one for logging successful login attempts and the other to document failed ones (`INFO` for success and `WARNING` for failures).

### 1. Setting Up Your Environment

Before starting, set up your virtual environment and install Flask:

```bash
python -m venv myenv

# For Mac
source myenv/bin/activate

#Install fastapi
pip install fastapi
```

### 2. Creating a Simple Flask Application

When you send a POST request to the **/login** endpoint with a username and password parameter, the server will check if the credentials are valid. If they are, the logger records the event using logger.info() to signify a successful login attempt. However, if the credentials are invalid, the logger records the event as a failed login attempt using logger.error().

```py
#Making Imports
from fastapi import FastAPI, request
import logging
import os

# Initialize the Fastapi app
app = Flask(__name__)
app = FastAPI()


# Configure logging
if not os.path.exists('logs'):
    os.makedirs('logs')
log_file = 'logs/app.log'
logging.basicConfig(filename=log_file, level=logging.DEBUG, format='%(asctime)s - %(levelname)s - %(message)s')
log = logging.getLogger(__name__)


# Define route and handler
@app.get('/login')
def login():
    log.info('Received login request')
    username = request.form['username']
    password = request.form['password']
    if username == 'admin' and password == 'password':
        log.info('Login successful')
        return 'Welcome, admin!'
    else:
        log.error('Invalid credentials')
        return 'Invalid username or password', 401

if __name__ == '__main__':
    app.run(debug=True)
```

### 3. Testing the Application

To test the application, run the Python script and access the **/login** endpoint using a web browser or a tool like curl. For example:

**Test Case 01**

```bash
 curl -X POST -d "username=admin&password=password" http://localhost:5000/login
```

**Output**

```bash
Welcome, admin!
```

**Test Case 02**

```bash
curl -X POST -d "username=admin&password=wrongpassword" http://localhost:5000/login
```

**Output**

```
Invalid username or password
```

**app.log**

```bash
2024-05-18 12:36:56,845 - INFO - Received login request
2024-05-18 12:36:56,846 - INFO - Login successful
2024-05-18 12:36:56,847 - INFO - 127.0.0.1 - - [18/May/2024 12:36:56] "POST /login HTTP/1.1" 200 -
2024-05-18 12:37:00,960 - INFO - Received login request
2024-05-18 12:37:00,960 - ERROR - Invalid credentials
2024-05-18 12:37:00,960 - INFO - 127.0.0.1 - - [18/May/2024 12:37:00] "POST /login HTTP/1.1" 200 -
```
