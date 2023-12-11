# Deploying your hobby project

My first serious production project was a website for D.S.A.V. Dodeka, the student athletics association I'm a member of. We wanted to implement a login system and for that we need to deploy code to a server. We also needed a database, so this would also have to run on this server. Furthermore, we have very little funds. We also already had a frontend and we made the choice to have a fully separate backend with a simple API. This describes my journey and can serve as a guide for those in a similar situation.

<!-- toc -->

## Assumptions

1. You want to deploy your application as an HTTP server that exposes endpoints to the internet
1. You want to deploy this server using a Docker container
    * You're storing your deployment in a Git repository
1. You want to automate the building of your application
1. Your application needs access to a few secrets (like a private key) that you do not want to permanently store in your database
1. You have access to a simple Linux server (we're going to assume its running a Debian-based sitro like Ubuntu)
1. You want to run and test in a few different environments (like local development, staging and production)
1. You want to run it alongside at least one external service, like a PostgreSQL database

## Optional assumptions

For the sake of this guide, we're going to make a few more assumptions that don't matter very much, but will make it simpler to explain things. 

1. Your deployment configuration and application are stored in a single Git repository
2. You have access to some kind of automations pipeline (like GitHub Actions or GitLab Pipelines)

This is not necessary if you have a way to import your application into your deployment repository.

## 1. Choosing a web server framework

First of all, you will need to choose a web server framework (if you're reading this, you probably already have). Any framework works, as long as it is able to accept HTTP connections on a port in localhost. We're going to assume it needs some secrets provided to it as environment variables, but nothing else. We assume it doesn't need any persistence (for that we're going to rely on a database).

In my case, I used FastAPI, a Python web framework designed primarily for building APIs. Again, anything works, it doesn't have to be Python.

Our demonstration server will have two routes:

* `/start/` will return a JSON value of `{'hello': 'deploy'}`
* `/secret/mysecreturl` will return the value of the environment variable `MY_SECRET`. 

The server will fail to start if `MY_SECRET` is not defined or has a length of zero.

## 2. Packaging your application as a Docker container

Next, we will need to package our application as a Docker container. We'll create the folder `/deploy/containers/app` and inside it a file with the name `Dockerfile`. This will be where we define our app container.

Our Dockerfile will look like this:

```Dockerfile
# We're going to run our application using Node 20 on a Debian (version 11, called 'bookworm') slim (so with very little pre-installed) container 
FROM node:20-bookworm-slim

# Here we copy the entrypoint that will be run when the container starts
COPY entrypoint.sh .

# Here you want to copy all your source files
COPY package.json .
COPY package-lock.json .
COPY app.js .

# Command to install dependencies
# This won't alter the package-lock.json
RUN npm ci

EXPOSE 8871

ENTRYPOINT ["./entrypoint.sh"]
```

And our entrypoint will look like this:
```bash
#!/bin/bash

# Define function to stop the appp gracefully
stop_app() {
    echo "Stopping app.."
    kill $app_pid
    exit 0
}

# Trap SIGTERM signal and run stop_app function
trap 'stop_app' SIGTERM

# 'node app.js' is the actual process we want to run!
node app.js &
app_pid=$!  # Save the PID of app process

wait "$app_pid"  # Wait for app
```

You might wonder why this entrypoint has all this bash magic. Without it, when stopping your container using an external trigger (like when we'll use Docker Compose) will not directly propagate, meaning it can take over 10 seconds to shut down. Using this simple pattern for your `entrypoint.sh` will make sure it shuts down instantly.


Whatever web framework or programming language you use, your Dockerfile will look similar:
- Copy source files
- Install dependencies using only the exact dependencies in the lockfile
- Define an entrypoint script

We're not going to ever build this container locally once everything is up and running, but let's test it our real quick (run this from the main directory of your repository):

```bash
# we run this just to make sure there's nothing left
rm -rf deploy/context
mkdir -p deploy/context
# let's not copy the node_modules file
cp app/*.js deploy/context
cp app/*.json deploy/context
cp deploy/containers/app/* deploy/context
cd deploy/context
docker build -t hellodeploy-app .
cd ../..
rm -rf deploy/context
```

If you don't have bash on your development machine (it should be available via Git Bash, because you definitely should have Git installed), all of those commands can be simply done manually. All you have to do is create a directory containing the Dockerfile, entrypoint.sh and the source files and dependency files of your application and then run `docker build -t hellodeploy-app .`.

Now, let's test the image:

```
docker run -it -e MY_SECRET=secretvalue -p 8871:8871 hellodeploy-app
```

If we navigate to `http://localhost:8871/secret/mysecreturl` we indeed get the response "secretvalue". Note that we provided the value of our secret as a command line argument to Docker.

### Docker Compose

We just deployed our application! ... except we don't want to manually run the above command and keep our terminal open. We also don't want to manually enter the secret on the command line like that. 

First, let's create a Compose file for Docker Compose, which will allow us to more easily start and stop our application and, later, add additional services.

We'll create a new folder, `/deploy/use/production` and add a `docker-compose.yml` inside of it. Let's slowly build up our Compose file.

```yaml
services:
  # Service name
  app:
    container_name: hellodeploy-app
    # Docker image that it will set up
    image: hellodeploy-app
    environment:
      MY_SECRET: ${MY_SECRET:?err}
    # This maps ports according to host_hostname:host_port:container_port
    ports:
      - "127.0.0.1:8871:8871"
```

Now, instead of `docker run`, we use:

```bash
MY_SECRET=secretvalue docker compose up
```

(Note we are still passing the environment variable directly. If you're not using bash as your shell, you can whatever alternative to like to set the environment variable, although we'll later be changing this completely)

We can run it in the background using (`-d` for detached):

```bash
MY_SECRET=secretvalue docker compose -p hellodeploy up -d
```

This time we're also specifying the project name (`-p hellodeploy`). This allows us to easily shut down our project using:

```bash
docker compose -p hellodeploy down
```

## 3. Build automation

Alright, we've successfully packaged our application and we can run it in a simple way! In fact, all we need is Docker Compose and the ability to set an environment variable.

However, we really don't want to do this manually. Let's automate things. I'll be assuming you're using GitHub Actions, but really any CI provider will work with only some minor modifications.

We'll create two files:
* `.github/workflows/ci.yml`
* `.github/workflows/app.yml`

In the future, we'll be having multiple actions, so let's create a single entrypoint action (`ci.yml`):

```yml
name: CI

on:
  push:
    # replace with the name of your main branch (e.g. master)
    branches: [ main ]

permissions:
  packages: write
  contents: read

jobs:
  build-app:
    strategy:
      matrix:
        target: [ 'staging' ]
    # replace tiptenbrink/hellodeploy with your own repository 
    uses: tiptenbrink/hellodeploy/.github/workflows/app.yml@main
    with:
      env: ${{ matrix.target }}
```

As you can see, we'll be using GitHub packages and their container registry, so we need to set the correct permissions for that. For now, the only environment we'll be building for is staging, so let's set that up in a matrix (so in the future we can easily add other environments).


Now the actual workflow:

```yml
name: App Build

on:
    # this allows it to be called from another workflow
    workflow_call:
      inputs:
          # this is the environment we're building for
          env:
            required: true
            type: string

permissions:
  packages: write
  contents: read

jobs:
  build-server:
    runs-on: ubuntu-latest
    steps:
      # we check out the repository
      - uses: actions/checkout@v4
      # we move the source and deploy files, just like we did before manually
      - name: Move source
        run: |
            mkdir -p deploy/context
            cp app/*.js deploy/context
            cp app/*.json deploy/context
            cp deploy/containers/app/* deploy/context
      # we log into the GitHub Container Registry (GHCR, alternative to Docker Hub)
      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor}}
          password: ${{ github.token }}
      # Set up buildx for later build-push-action, which is necessary for cache feature
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      # Build and push
      - name: Build and push
        uses: docker/build-push-action@v3
        with:
          # where we moved all the files to that will be used to build the image
          context: deploy/context
          # replace tiptenbrink with your organization name or GitHub username (i.e. the parent name of your repo)
          tags: ghcr.io/tiptenbrink/hellodeploy-app:${{ inputs.env }}
          # we use the GitHub Actions (gha) cache to make future builds faster
          cache-from: type=gha
          cache-to: type=gha, mode=max
          push: true
```

We're not doing anything new, just what we did before but now utilizing some external actions (only those officially provided by GitHub and Docker!). 

Once you push this to your main branch on GitHub, if everything goes right GitHub will now build and publish your images as "packages". 

If you want to use these images locally, you have to log in to the GitHub container registry, using:

`docker login ghcr.io`

Once you're logged in, we can change our Compose file (be sure to replace tiptenbrink with the correct name as you did in `app.yml`):

* ~~image: hellodeploy-app~~ -> image: ghcr.io/tiptenbrink/hellodeploy-app:production
* ~~container_name: hellodeploy-app~~ -> container_name: hellodeploy-app-production

If we start up Docker Compose now, it will first pull the image from the GitHub Container Registry and then start it up. 

It's quite possible you don't need anything more than this if you're fine with just passing in a few secrets and running the Compose commands manually. Complexity will increase significantly in the following sections, but I think it's worth it! Do check out the section on the additional external service, as you almost certainly will need a database.

## 4. Secrets management

This will probably be the most opiniated part of this guide, but I would recommend you follow a similar idea, although the implementation might differ significantly based on which platform you choose.

Previously, when we ran `docker compose up`, we had to ensure our environment variable was already set to our secret value. When we have many of them, this is not very practical. Furthermore, we'd rather not save them in our shell history. We want this to happen out of sight and encrypted as much as possible.

I hope that if you're reading this, you know the value of a password manager, or at least some way to securely store and access your passwords. The same should hold for the secret values used in your application. Thankfully, Bitwarden, an excellent open source password manager, recently launched Bitwarden Secrets Manager. This product is also open source and you can self-host it, but I recommend using their hosted managed free tier (if you need more users with access to secrets, I assume you have budget to pay for their Teams tier). 

I don't like relying on free tiers (although with GitHub of course everyone is also relying on this), because companies often change them once users are locked in. However, Bitwarden's API is very simple and would be easy to replace (and you can always choose to self-host for free). The company also has a good track record. So, in this guide I will assume you're using Bitwarden Secrets Manger.

In the main part of this guide I will not be using any of my own tools, but I do suggest to check out the recommended sections, particularly the one about `tidploy`.

### Deployer container

(If you're using `tideploy` and Bitwarden Secrets Manager, you won't need to build your own deployer image)

Our main goal is to minimize the use of complex bash scripts and maximize our direct use of Docker. So, we will be using a Docker container as a sort of script to collect our secrets and load them as environment variables, before actually running our applciation.

This works as follows (inspired by [this StackOverflow question](https://stackoverflow.com/questions/32163955/how-to-run-shell-script-on-host-from-docker-container)):

- We install the secrets manager and dependencies (like Rust) only in our "deployer" container
- We call this container, providing it only with an access token
- The container loads the secrets from our secrets manager and then passes them back to the host through a named pipe
- The host calls our entrypoint, which starts up Docker

Here's our new Dockerfile, which I'm putting in `deploy/containers/`:

```Dockerfile
FROM rust:1-slim-bookworm

RUN cargo install bws -y
RUN cargo install nu -y

WORKDIR /dployer

COPY entrypoint.nu .

ENTRYPOINT ["./entrypoint.nu"]
```

We're installing two Rust programs, Bitwarden Secrets Manager and Nushell. I'm writing the entrypoint in Nu because it provides some nice out-of-the-box ways to deal with JSON (which is output by Bitwarden Secrets Manager). Of course, if you're more comfortable with `jq` and just bash, definitely use that instead. I do think the Nushell code is very understandable.

Here's the `entrypoint.nu` ([link with syntax highlighting](https://github.com/tiptenbrink/hellodeploy/blob/main/deploy/containers/deployer/entrypoint.nu)):

```nu
#!/usr/bin/env nu
def secret_to_pipe [secret: string, pipe: string] {
    # we request the secret from Bitwarden Secret Manager using its id, loading it as JSON
    let j = bws secret get $secret | from json
    # the key is identical to the environment variable key
    let k = $j | get key
    # the value is the actual secret
    let v = $j | get value
    # for each secret we append a newline and <KEY>=<VALUE> to the named pipe
    echo $"\n($k)=($v)" | save $pipe --append
}

def from_deploy [file, pipe: string] {
    # open the file and load it as Nu's JSON representation
    let j = open $file --raw | decode utf-8 | from json
    # we get the value of secrets.ids, which is an array of id values
    let secrets = $j | get secrets | get ids
    # we call the secret_to_pipe function for each id and also add the name of the pipe as an argument
    for $e in $secrets { secret_to_pipe $e $pipe }
}

# main entrypoint
def main [] {
    # we call the from_deploy function with arguments tidploy.json and ti_dploy_pipe
    from_deploy tidploy.json ti_dploy_pipe
    # once we're done we append a newline and TIDPLOY_READY=1 to the named pipe
    echo "\nTIDPLOY_READY=1" | save ti_dploy_pipe --append
}
```

So what is this named pipe? Well, we're going to have to create it. It's a very simple FIFO pipe located in a file. It's what we will use to communicate between the Docker container and host.

## Security best practices

## Recommended: tidploy

## Optional: configuration management

## Optional: command shortcuts

## Appendix A: dependencies

### Required server dependencies

(We're assuming all the basic dependencies on a standard Ubuntu image are installed, like a modern version of Python, git, bash, a C library, etc.)

* `Docker Engine` and `Docker Compose`
* That's it!

### Recommended

* `bws` (Bitwarden Secrets Manager CLI)
* `tidploy`
* To install `tidploy` and `bws`, you probably want to have a minimal Rust toolchain installed, which will include `cargo`. Howeve, by default cargo will compile from source, which can take quite long on small VMs, so binary installation is the easiest solution. To install these programs easily, do:
    * `cargo install cargo-quickinstall`
    * `cargo quickinstall cargo-binstall`
    * `cargo bininstall <tool>` (where `<tool>` can be `bws` and `tidploy`)

