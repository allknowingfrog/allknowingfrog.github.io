---
layout: post
title:  "Rails Development via Docker Compose"
date:   2017-10-15 04:00:00 -0600
---

I've wrestled a lot with [Docker Compose](https://docs.docker.com/compose). It feels like too much added complexity for simple environments, and it seems to have some quirks that don't come up with raw Docker commands. However, I recently buckled down and filled some gaps in my knowledge, and I think I'm switching to Docker Compose for Rails development.

I generally launch my Rails container with an interactive bash terminal, which lets me think of it as a remote server. I run commands, like setting up the database, and then start up the Rails development server (Puma) when I'm ready. However, the Docker Compose environment needs a non-interactive and ongoing process, i.e. the development server should be the primary command. The current Rails version actually throws an error if you try to start up Puma without setting up a database, so launching directly into the development server isn't an option either. In the end, I put together a whole startup script.
```
#!/bin/bash
cd /var/www/html
rm tmp/pids/server.pid || true
mailcatcher --http-ip=0.0.0.0
wait-for-it -t 0 db:3306
rails db:setup
rails s
```
This cleans out the file Puma uses to track its PID, as this sometimes gets left behind from previous runs and causes Puma to refuse to launch "another" instance. It also launches [Mailcatcher](https://mailcatcher.me) and then waits for the database container to initialize with [wait-for-it](https://github.com/vishnubob/wait-for-it). This was another obstacle to overcome, as Docker Compose can control the order in which containers start, but cannot wait for one to be "ready" before launching another.

The aforementioned script (`dockerup.sh`) lives in the root directory of the Rails project alongside `Dockerfile`, which makes it available as an executable and loads configuration files from the root directory.
```
FROM ruby:2.4

RUN gem install rails

RUN apt-get -y update && apt-get install -y nodejs

RUN gem install mailcatcher

RUN mkdir -p /var/www/html

WORKDIR /var/www/html

VOLUME /var/www/html

RUN git clone https://github.com/vishnubob/wait-for-it && \
    mv wait-for-it/wait-for-it.sh /usr/local/bin/wait-for-it

ADD dockerup.sh /usr/local/bin/dockerup

RUN chmod +x /usr/local/bin/dockerup

ADD Gemfile /var/www/html/Gemfile

ADD Gemfile.lock /var/www/html/Gemfile.lock

RUN bundle install
```
Running `bundle install` here is important. Unlike the database setup, installed Gems can become part of the Docker image and save a lot of time when building containers from existing images.

Finally, `docker-compose.yml` (also in the project root) ties it all together with a database from the default MySQL image (or PostgreSQL, or whatever).
```
version: "3.3"
services:
  web:
    build: .
    networks:
      - default
    ports:
      - "3000:3000"
      - "1080:1080"
    volumes:
      - .:/var/www/html
    command: dockerup
    depends_on:
      - db
  db:
    image: mysql:latest
    networks:
      - default
    environment:
      MYSQL_ROOT_PASSWORD: guest
networks:
  default:
    driver: bridge
```
From this point, launching the whole stack is as simple as `docker-compose up`. Perhaps more importantly, `docker-compose down` completely destroys it, making for easy rebuilds of the environment. With raw Docker commands, I find myself keeping the same containers alive for long periods of time to avoid the inconvenience of rebuilding them. With Docker Compose, I rebuild every time I start work on a project.

As a final thought, I don't think I lose much by getting away from the interactive bash terminal. I can always use `docker-compose exec` if I really want one, but I mostly need to run generators and migration commands, which can be accomplished with `docker-compose run`.
