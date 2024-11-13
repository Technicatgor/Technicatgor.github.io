---
layout: post
title: "Simple Nginx proxy pass with frontend and backend"
date: 2024-11-12 16:00:00 +800
categories: [Networking, ReverseProxy]
tags: [nginx, vuejs, fastapi, python, vite]
image:
  path: /assets/img/headers/simplenginx.png
  lqip: data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAA8gAAAH4CAIAAAAZ1VPRAALJIklEQVR4Aeyah5IbuxVE0aQ3h/eUv8L//1MOm/NumxaKp9TVZInO4e1d1QgDXNyMHgyG+vj59+Od
---

## Diagram

In my scenario, I am using the traditional method to deploy the Vue application. I set up an Nginx server in a container and map the build files of the Vue app to the Nginx HTML volume. The server is configured to proxy requests to a specific path, `/form`. Additionally, I will deploy FastAPI in a separate container, which will also use proxying for requests directed to the path `/api`.
![/assets/img/nginx_flow.png](/assets/img/nginx_flow.png)

## Nginx

Make certs folder and generate ssl certificates first.

```
mkdir certs
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout ./certs/example.key -out ./certs/example.crt
```

Create `nginx.conf`.

```conf
upstream fastapi {
    server 192.168.50.245:8000
}

server {
    listen 80;
    server_name test.example.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl;
    server_name test.example.com;

    ssl_certificate /etc/nginx/ssl/example.crt;
    ssl_certificate_key /etc/nginx/ssl/example.key;

    location / {
        root /var/www/html;
        index index.html index.htm;
    }

    location /api {
        proxy_pass http://fastapi;
    }
}
```

Create `docker-compose.yml`

```yml
services:
  nginx:
    image: nginx:latest
    ports:
      - 80:80
      - 443:443
    volumes:
      - ./html:/var/www/html
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
      - ./certs:/etc/nginx/ssl
```

Create a html for the main page.
`mkdir html && vim html/index.html`

```html
<h1>Main Page</h1>
```

## FastAPI

Install fastapi and uvicorn
`pip install fastapi uvicorn`

Create `main.py`

```py
from fastapi import FastAPI, APIRouter
from pydantic import BaseModel
from fastapi.responses import JSONResponse
from fastapi.middleware.cors import CORSMiddleware

# root_path use for behind the proxy
app = FastAPI(root_path="/api")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # Change this to specific origins in production
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Define a Pydantic model for the incoming data
class UserEntry(BaseModel):
    fname: str
    lname: str
    age: int

@app.post("/user")
async def create_entry(user_entry: UserEntry):
    user_data = user_entry.dict()
    # Here you can process the data (e.g., save it to a database)
    # For now, we will just return the received data as a response
    return JSONResponse(content={"message": "User entry received", "data": user_data})

```

Then export your `requirements.txt`, `pip freeze >> requirements.txt`

Prepare a `Dockerfile` for running in docker

```dockerfile
FROM python:3.10

# Set the working directory in the container
WORKDIR /app

# Copy the code & requirements file into the container
COPY ./requirements.txt .
COPY ./main.py .

# Install the dependencies specified in requirements.txt
RUN pip install --no-cache-dir --upgrade -r /app/requirements.txt

# Command to run the FastAPI application using Uvicorn
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Create a `docker-compose.yml` for easy to deploy.

```yml
services:
  fastapi:
    container_name: fastapi
    build: .
    ports:
      - 8000:8000
    restart: unless-stopped
```

## Vue app

Initial vue app by vite \
`npm create vite@latest my-vue-app -- --template vue`

`package.json`

```json
{
  "name": "vue-app",
  "private": true,
  "version": "0.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite --host 0.0.0.0",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "axios": "^1.7.7",
    "vue": "^3.5.12"
  },
  "devDependencies": {
    "@vitejs/plugin-vue": "^5.1.4",
    "vite": "^5.4.10"
  }
}
```

Install node modules with using `npm i`

In `vite.config.js`, add a base url section is needed, we can import from a .env.production file as well.

```js
import { defineConfig } from "vite";
import vue from "@vitejs/plugin-vue";

// https://vite.dev/config/
export default defineConfig({
  base: "https://test.example.com/form",
  plugins: [vue()]
});
```

Edit `App.vue`

```vue
<script setup>
import UserForm from "./components/UserForm.vue";
</script>
<template>
  <UserForm />
</template>
<style>
body {
  font-family: Arial, sans-serif;
  background-color: #fff;
  margin: 0;
  padding: 20px;
}
</style>
```

Create `./components/UserForm.vue`

```vue
<template>
  <div>
    <h3>User Entry Form</h3>
    <form @submit.prevent="submitEntry">
      <div>
        <label for="fname">First Name:</label>
        <input type="text" v-model="form.fname" id="fname" required />
      </div>
      <div>
        <label for="lname">Last Name:</label>
        <input type="text" v-model="form.lname" id="lname" required />
      </div>
      <div>
        <label for="age">Age:</label>
        <input type="number" v-model="form.age" id="age" required />
      </div>
      <button type="submit">Submit</button>
    </form>
    <p v-if="loading">Submitting...</p>
    <p v-if="user_response">{{ user_response }}</p>
  </div>
</template>

<script>
import axios from "axios";

export default {
  data() {
    return {
      form: {
        fname: "",
        lname: "",
        age: ""
      },
      loading: false,
      user_response: null
    };
  },
  methods: {
    async submitEntry() {
      this.loading = true;
      try {
        const res = await axios.post(
          "https://test.example.com/api/user",
          this.form
        );

        this.user_response = res.data; // Handle the response from the server
      } catch (error) {
        console.error(error);
        this.user_response = "Error submitting data";
      } finally {
        this.loading = false;
      }
    }
  }
};
</script>
<style scoped>
h3 {
  color: #000;
  text-align: center;
}
form {
  background-color: #fff;
  border-radius: 8px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
  max-width: 400px;
  margin: auto;
  padding: 20px;
}
div {
  margin-bottom: 15px;
}
label {
  display: block;
  margin-bottom: 5px;
  color: #000;
  font-weight: bold;
  text-align: left;
}
input[type="text"],
input[type="number"] {
  width: calc(100% - 10px);
  padding: 8px;
  border-radius: 4px;
  border: 1px solid #ccc;
}
input[type="text"]:focus,
input[type="number"]:focus {
  border-color: #007bff; /* Change this color as needed */
button {
  background-color: #007bff; /* Primary button color */
  color: white;
  border: none;
  border-radius: 4px;
  padding: 10px;
  cursor: pointer;
  width: 100%;
}
button:hover {
  background-color: #0056b3; /* Darker shade on hover */
}
p {
  color: #000;
  text-align: center;
}
</style>
```

Build the vue app with using `npm run build`, you will get the `dist` build files

```bash
.
├── dist
├── index.html
├── node_modules
├── package.json
├── package-lock.json
├── public
├── README.md
├── src
└── vite.config.js
```

Then we copy the context of dist to map nginx HTML volume, \
`cp -r vue-app/dist/* nginx/html/form/.`

## Docker Compose

```bash
docker-compose up -d -f nginx/docker-compose.yml -f fastapi/docker-compose.yml
```

## DNS record

Add a hostname resolve for accessing the domain in locally. \
`echo "192.168.50.245 test.example/com" >> /etc/hosts`
Now you can go to `https://test.example.com/form` on the browser.

## Conclusion

For CICD, I recommend to deploy a container with vue app as well. Then compose all the services in `docker-compose.yml`
I create a Dockerfile for vue app:

```dockerfile
FROM node:lts-alpine as build-stage
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# production stage
FROM nginx:stable-alpine as production-stage
COPY --from=build-stage /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

`docker-compose.yml`

```yml
services:
  vue-app:
    container_name: vue-app
    build: .
    ports:
      - 3000:80
    restart: unless-stopped
```

In nginx proxy configuration:

```conf
upstream vue-app {
    server 192.168.50.245:3000
}
server {
  location /form {
      proxy_pass http://vue-app/;
  }
}
```
