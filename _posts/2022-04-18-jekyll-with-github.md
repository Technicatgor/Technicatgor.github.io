---
layout: post
title: "Meet Jekyll - The Static Site Generator"
date: 2022-04-18 10:00:00 -0500
categories: self-hosted
tags: jekyll websit

---

Jekyll is a static site generator. It takes text written in your favorite markup language and uses layouts to create a static website. You can tweak the site’s look and feel, URLs, the data displayed on the page, and more.

## Instructions
- Ruby version 2.5.0 or higher, including all development headers (check your Ruby version using ruby -v)
- RubyGems (check your Gems version using gem -v)
- GCC and Make (check versions using gcc -v,g++ -v, and make -v)

[References](https://jekyllrb.com/docs/installation/)

Install all prerequisites.

Install Dependencies

```sh
sudo apt update
sudo apt install ruby-full build-essential zlib1g-dev git 

echo '# Install Ruby Gems to ~/gems' >> ~/.bashrc
echo 'export GEM_HOME="$HOME/gems"' >> ~/.bashrc
echo 'export PATH="$HOME/gems/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```
Install Jekyll `bundler`

```sh
gem install jekyll bundler
```

### Create a site based on Chirpy theme

git fork with [jekyll-theme-chirpy](https://github.com/cotes2020/jekyll-theme-chirpy)

After forked, clone your repo and install dependencies	

```sh
git clone git@<YOUR-USER-NAME>/<YOUR-REPO-NAME>.git
cd repo
bundle 
```

After making changes to your site, commit and push then up to git
```sh
git add .
git commit -m "after change"
git push 
```
### Jekyll Command 

```sh
bundle exec jekyll s
```

Building your site in production mode
```sh
JEKYLL_ENV=production bundle exec jekyll b
```
This will output the production site to `_site`


### Building Site in CI

This site already works with GitHub actions, just push it up and check the actions Tab.,

For GitLab you can see the pipeline I built for my own docs site here

### Building with Docker

Create a `Dockerfile` with the following
```dockerfile
FROM nginx:stable-alpine
COPY _site /usr/share/nginx/html
```
Build site in production mode
```sh
JEKYLL_ENV=production bundle exec jekyll b
```
Then build your image:
`docker build .`

### Create a Post 

Naming Conventions

Jekyll uses a naming [convention for pages and posts](https://jekyllrb.com/docs/posts/)

Create a file in _posts with the format

`YEAR-MONTH-DAY-title.md`

For example:

`2022-05-23-homelab-docs.md`
`2022-05-34-hardware-specs.md`


