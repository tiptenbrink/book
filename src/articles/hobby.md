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
1. (Non-essential) You want to run and test in a few different environments (like local development, staging and production)
1. (Non-essential) You want to run it alongside at least one external service, like a PostgreSQL database

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
# We're going to run our application using Node 20 on a Debian (version 11,
# called 'bookworm') slim (so with very little pre-installed) container 
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
      MY_SECRET: ${MY_SECRET_VAR:?err}
    # This maps ports according to host_hostname:host_port:container_port
    ports:
      - "127.0.0.1:8871:8871"
```

Now, instead of `docker run`, we use:

```bash
MY_SECRET_VAR=secretvalue docker compose up
```

(Note we are still passing the environment variable directly. If you're not using bash as your shell, you can whatever alternative to like to set the environment variable, although we'll later be changing this completely)

We can run it in the background using (`-d` for detached):

```bash
MY_SECRET_VAR=secretvalue docker compose -p hellodeploy up -d
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
  build-app:
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
      # we log into the GitHub Container Registry (GHCR, alternative to 
      # Docker Hub)
      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor}}
          password: ${{ github.token }}
      # Set up buildx for later build-push-action, which is necessary for 
      # cache feature
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      # Build and push
      - name: Build and push
        uses: docker/build-push-action@v3
        with:
          # where we moved all the files to that will be used to build 
          # the image
          context: deploy/context
          # replace tiptenbrink with your organization name or GitHub username
          # (i.e. the parent name of your repo)
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
# Slim Debian container with Rust toolchain (including the cargo package 
# manager) installed
FROM rust:1-slim-bookworm

# to easily download binaries, we need curl. we don't want our container to be 
# too large so we delete the cache afterwards
RUN apt-get update \
	&& apt-get install -y curl \
	&& rm -rf /var/lib/apt/lists/* \
	&& rm -rf /var/cache/apt/*

# quickinstall is easy to compile
RUN cargo install cargo-quickinstall
# we then get cargo-binstall as a binary using quickinstall
RUN cargo quickinstall cargo-binstall
# we can now easily get these as binaries also
RUN cargo binstall bws -y
RUN cargo binstall nu -y

WORKDIR /dployer

COPY entrypoint.nu .

ENTRYPOINT ["./entrypoint.nu"]
```

We're installing two Rust programs, Bitwarden Secrets Manager and Nushell ([check out the latter if you haven't already!](https://www.nushell.sh/)). I'm writing the entrypoint in Nu because it provides some nice out-of-the-box ways to deal with JSON (which is output by Bitwarden Secrets Manager) and can easily utilize parallelism. Of course, if you're more comfortable with `jq` and just bash, definitely use that instead. I do think the Nushell code is very understandable.

Here's the `entrypoint.nu` ([link with syntax highlighting](https://github.com/tiptenbrink/hellodeploy/blob/main/deploy/containers/deployer/entrypoint.nu)):

```nu
#!/usr/bin/env nu
def secret_to_pipe [secret: string] {
    # we request the secret from Bitwarden Secret Manager using its id, loading it as JSON
    let j = bws secret get $secret | from json
    # the key is identical to the environment variable key
    let k = $j | get key
    # the value is the actual secret
    let v = $j | get value
    # for each secret we append a newline and <KEY>=<VALUE> to the named pipe
    return $"\n($k)=($v)"
}

def from_deploy [file, pipe: string] {
    # open the file and load it as Nu's JSON representation
    let j = open $file --raw | decode utf-8 | from json
    # we get the value of secrets.ids, which is an array of id values
    let secrets = $j | get secrets | get ids
    # we call the secret_to_pipe function for each id in parallel and join them 
    # with new lines
    let output = $secrets | par-each { |e| secret_to_pipe $e } | str join "\n"
    # the result we send to the pipe
    echo $output | save $pipe --append
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

First though, we will also want to build our deployer container automatically. This is almost an exact copy-paste of the previous GitHub Actions workflow file, so I won't repeat it here. [Check out the repository](https://github.com/tiptenbrink/hellodeploy/blob/8ad6c682a609cb3872900543b660b4d57e870ba6/.github/workflows/deployer.yml) if you're unsure.

### Bitwarden Secrets Manager

Before we move on, you will now need to store the "secretvalue" in Bitwarden Secrets Manager. [Here's a guide](https://bitwarden.com/help/secrets-manager-overview/) on how to use it. You will need to make an organization and subscribe to secrets manager (which is free). You'll need to log in to the web version and then you can open Secrets Manager through the product switcher in the top right (the little app grid icon). Then, you can create a project and add secrets to it. 

I recommend making their name identical to the environment variable you will pass into Docker Compose (in our case `MY_SECRET_VAR`). If you don't do that, you will need to modify the `entrypoint.nu` script or the script we're going to make in the next section to ensure it has the right name before being passed to Docker Compose.

Finally, you'll want to make a "service account" and give it read permission to the project that contains your secrets. Once you've done that, you can generate an access token. Keep this access token very safe! As safe as you would the master password of your password manager, as it gives access to all secrects the associated service account can read. In the main part of this tutorial, we won't exactly be keeping it very safe, but do check out the recommended sections as well for some guidance.

You'll also want to get the "ids" of the secrets you created. These are random identifiers that are safe to show publicly as you will still need the access token to actually retrieve the secrets. Note that with the access token you can easily access these id's so there's no benefit to securing them. Let's assume the id of `MY_SECRET_VAR` is "abcdef01-23fe-dcba-4567-abcd89876543".

### Deploy unit

Now we'll return to our actual deployment, so let's go back to the `deploy/use/production` folder. Remember, we're going to use our deployer container as a script to get our secrets for us. However, as you might have seen, the `entrypoint.nu` needs a file called "tidploy.json" which we haven't  provided to it. It will be this file that is going to contain the id's of our secrets.

`tidploy.json` is going to have a certain structure that will align with our `entrypoint.nu` script:

```json
{
    "secrets": {
        "ids": [
            "abcdef01-23fe-dcba-4567-abcd89876543"
        ]
    }
}
```

Finally, we'll need two more scripts. First, the script that will read the secrets and start our server (`source.sh`):

```bash
#!/bin/bash
# we want to auto-export all environment variables we set so docker compose can use them
set -a
echo "Waiting for secrets..."
while [ true ] 
do 
    # if file exists and is named pipe
    if [ -p "$1" ]; then
        . $1
        if [ -n "$TIDPLOY_READY" ]; then
            echo "Starting...."
            # ensure we have the latest version of our images
            docker compose pull
            # here we start our compose file as before
            docker compose -p hellodeploy up -d
            break
        else
            echo "Secrets loaded."
        fi
    # if pipe doesn't exist we don't want to run too many loops
    else
        sleep 1
    fi
done
```

And then, to tie it all together, our deployer script (which we will call `dployer.sh`):

```bash
#!/bin/bash

cleanup() {
    echo
    echo "Removing pipe..."
    rm -f deploypipe
    exit 1
}

# if you do Ctrl+C it will run cleanup
trap cleanup SIGINT

# remove the pipe if it somehow still exists
rm -f deploypipe
# create a named fifo pipe at ./deploypipe
mkfifo deploypipe
# run the deployer, providing it with the Secrets Manager access token and
# mounting the named pipe as well as the JSON containing the secrets to the 
# container
# be sure to replace ghcr.io/tiptenbrink/hellodeploy-deployer:latest with the 
# location of your own container
# the process is started in the background (the '&') and then we run our 
# previous script with the name of the pipe as the first argument
docker run \
  -e BWS_ACCESS_TOKEN \
  -v ./deploypipe:/dployer/ti_dploy_pipe \
  -v ./tidploy.json:/dployer/tidploy.json \
  ghcr.io/tiptenbrink/hellodeploy-deployer:latest & \
./source.sh ./deploypipe
# finally we clean up the pipe by removing it
rm deploypipe
```

That's it! Let's get to deploying it to our server.

## 5. Linux server

Now that we have all our files ready and our images built, we want to get things running on our server. I am going to assume you have some way of exposing the port of your app to the outside world (e.g. with an nginx reverse proxy). Be sure you've already set up all of that before this step! Other than that, all you will need on your server is Git, Docker (with Docker Compose) and bash.

First, we're going to check out our repository. This might be immediately problematic if your repository has a very long history or contains a lot of large assets. If this is a problem, refer to the optional section [about partial clones and sparse checkout](#optional-partial-clone-and-sparse-checkout).

Once you have the repository checked out on your server, let's navigate to the `deploy/use/production` folder, which includes our Docker Compose file, the two scripts and our JSON file with the secret ids:

```
deploy
├── containers
│   └── ...
└── use
    └── production
        ├── docker-compose.yml
        ├── dployer.sh
        ├── source.sh
        └── tidploy.json
```

Then, all we have to do is run `BWS_ACCESS_TOKEN=<your access token> ./source.sh`.

But wait... how is that better than what we did at the start, when we just said `MY_SECRET_VAR=secretvalue docker compose -p hellodeploy up -d`. We're still putting a secret right into the command line. In fact, our access token is even more sensitive than just the one secret! So why did we put in all this effort?

The first reason is that if we have more than one secret, we still have to only pass in the access token, the rest will be done by our script. The second, less satisfying reason is, from now on simple shell scripts won't save you. I'll suggest three alternatives to passing in the access token. We'll discuss method 3 in the recommended section on [tidploy](#recommended-tidploy).

1. Store the access token in a file on your server, maybe only accessible to a specific user, with the format `MY_SECRET_VAR=secretvalue`. Before running the script, you then simply run `. <my secret file>` to load the environment variable (you might need to run `set -a` first to make sure it's exported). I don't recommend this, but it's better than just passing it in each time over the command line. NOT RECOMMENDED.
1. Run your script remotely with automations. For example, you could set up a GitHub Action that stores your access token safely in the GitHub environment, which then SSH's into your server (with the token loaded into the environment) and runs the `source.sh` script. This is one of the best solutions, but it requires setting up remote access and running automated scripts over SSH is not my favorite. I like being the one who actually runs it. 
1. Use an external program to store your secret in the OS' keyring/keychain and load it when you wan to run the script. This is my recommended solution and the one I will discuss, but it does require writing an actual CLI program (or using somebody elses) that can leverage the OS keyring APIs. Two libraries I recommend for this (one of which I will be using) are [keyring (Python)](https://github.com/jaraco/keyring) and [keyring-rs (Rust)](https://github.com/hwchen/keyring-rs). This means you can ensure your access token is never persistently stored in plaintext. See the recommended section on [tidploy](#recommended-tidploy) for more details.

So, I recommend using either method 2 or 3. If you're okay with an external dependency or if you don't mind writing your own CLI applications, I recommend 3.

That's it for the essential portion of this guide! Remember you can easily shut down using `docker compose -p hellodeploy down` from anywhere on your server. Also note that if you re-run the script (if container names and the project name are the same), Docker is smart enough to recreate the images if there's a newer version. If the images are the same, Docker will simply keep them running! The script is therefore (somewhat, as it does depend on external services like GHCR and Bitwarden Secrets Manager) idempotent. 

## 6. External service

Let's discuss how to add a PostgreSQL database to our application. We'll set up a new app, `app-multi`. I won't write the full source code here, but we'll add the following routes:

- `/db/` will query the database, returning "Helloy deploy!". 
- `/mode/` will return the current environment (which we'll introduce in the next section), which it reads from the `APP_MODE` environment variable.

The app connect to the default "postgres" database as user "postgres", using the password and port set by `POSTGRES_PASSWORD` and `POSTGRES_PORT`, respectively. If the environment is "localdev" it will connect using localhost, while otherwise it will use "hellodeploy-db-\<environment\>" (where environment is the value of the APP_MODE variable) as the host name (more on the latter later).

You can see the source code here []

### Database container

For demonstration purposes (to show you how you can change settings yourself), we'll use a custom `pg_hba.conf` (host-based authentication) and `postgresql.conf`. The `pg_hba.conf` is set to require a password and allow connections from all hosts. The database will listen on all addresses and use port 3141 instead of the default. We do this so that we can't just use the default on everything, to demonstrate that we have indeed set up our environment variables correctly!

Our Dockerfile then looks as follows:

```Dockerfile
FROM postgres:16-bookworm
ADD postgresql.conf /conf_dir/postgresql.conf
ADD pg_hba.conf /conf_dir/pg_hba.conf
CMD ["postgres", "-c", "config_file=/conf_dir/postgresql.conf"]
```

See the repository here [] for the contents of our configuration files. 

Again, we want to set up actions to build these automatically in our repository. These will be almost identical to the previous `app.yml`, so I won't go into detail here. See the repository for details [].

### Docker Compose file

Let's dive straight into it:


```yaml
services:
  db:
    container_name: hellodeploy-db-production
    image: ghcr.io/tiptenbrink/hellodeploy-db:production
    # A volume is basically a file system shared between the host and 
    # the container
    volumes:
      # <volume name in volumes below>:<container destination directory>
      # This means the volume will be accessible from the container in the 
      # dest. directory in the container
      - hellodeploy-db-volume:/hellodeploy-db
    environment:
      - PGDATA=/hellodeploy-db
      - POSTGRES_PASSWORD
      - POSTGRES_USER
    ports:
      - "127.0.0.1:3141:3141"
  
  app-multi:
    depends_on:
      - db
    container_name: hellodeploy-app-multi-production
    image: ghcr.io/tiptenbrink/hellodeploy-app-multi:production
    environment:
      MY_SECRET: ${MY_SECRET_VAR:?err}
      # we're okay if they're empty so we set default empty values
      POSTGRES_PASSSWORD: ${POSTGRES_PASSSWORD:-}
      POSTGRES_PORT: ${POSTGRES_PORT:-}
      APP_MODE: ${APP_MODE}
    ports:
      - "127.0.0.1:8871:8871"
volumes:
  # This name should correspond to the volume mentioned in volumes above
  hellodeploy-db-volume:
    # A local directory
    driver: local
    name: hellodeploy-db-volume-production
networks:
  # Set up a network so that other containers on this network can access
  # each other. A different container on this network can access this container
  # by using the container name as the hostname
  # So localhost:3000 is exposed to other containers as <container name>:3000
  default:
    name: hellodeploy
```

The network and volume additions are the most important. We want our database to not remove its data when the container stops, so we need some kind of persistence. For that, we need to use the host machine. That's why we use Docker volumes, which will remember the data put into them by the container. We mount this at `/hellodeploy-db` in the container and tell Postgres (by the PGDATA variable) that this is where the database should store its data.

The shared network allows communication between the containers. Remember we set the hostname used by our new app to connect to the database to be `hellodeploy-db-production` (when we're in the production environment, which is always the case right now). This is possible because they are on the same network. 

### Env file

To keep the Docker Compose as general as possible, I suggest to keep as many runtime options a .env file as possible. In our case, we have POSTGRES_PORT and APP_MODE. Let's define a file called "production.env" in our `deploy/use/production` as follows:

```bash
POSTGRES_PORT=3141
APP_MODE=production
```

To make sure this is loaded by Docker Compose, let's modify our `source.sh` slightly:

* ~~docker compose pull~~ `docker compose --env-file production.env pull`
* ~~docker compose -p hellodeploy up -d~~ `docker compose --env-file production.env -p hellodeploy up -d`

### Deploying

Thanks to our efforts in the previous section, deploying works exactly the same! Simply run `./dployer.sh`, but now using the modified files in `deploy/use/production`. Of course, we do have to add the new secret (the `POSTGRES_PASSWORD`) to Bitwarden Secrets Manager and add its id to our `tidploy.json`. Other than that, we are done! 

## 6. Different environments

Imagine your staging environment has a few different variables from your production environment. Your local development setup is probably even more different! How can we solve this?

Well, it's quite easy. Remember, previously we only made the `deploy/use/production` directory. Let's set up a staging environment

## Security considerations

## Recommended: tidploy

## Optional: without deployer container

If you don't like the container solution with named pipes to externalize the secret loading logic from your server, it's pretty easy to just do this on your server directly. That does mean you will have to install a Rust toolchain, `nu` (except if you write your own script) and `bws` (the Bitwarden Secrets Manager CLI).

`source.sh` becomes simply:

```bash
#!/bin/bash
set -a
secrets=$(nu secrets.nu)
eval "$secrets"
docker compose pull
docker compose -p hellodeploy up -d
```

`secrets.nu` is the modified version of our previous `entrypoint.nu` that we ran in the container. It's a small modification, which you can find on the repository [here](https://github.com/tiptenbrink/hellodeploy/blob/8ad6c682a609cb3872900543b660b4d57e870ba6/deploy/use/production_no_container/secrets.nu).

## Optional: partial clone and sparse checkout

To minimize the amount of files downloaded to your server when getting the deploy script. You can use two modern Git features: partial (and sparse) clone and sparse-checkout. Here is what you want to run:

```
git clone https://github.com/tiptenbrink/hellodeploy.git sparsedeploy --sparse --filter=tree:0
cd sparsedeploy
git sparse-checkout init --cone
git sparse-checkout set deploy/use/production
```

In the first command we're doing a clone, but we add the `--sparse` and `--filter=tree:0` options. The first one will only clone files in the root directory (so make sure there's not too many!) while the second will not download any trees and blobs from previous commits. It _will_ download commit data, but this is recommended so you can still access previous commits if you want to (it will then download the necessary objects on demand).

The second git command will initialize sparse-checkout mode and set it to "cone" mode. This is generally recommended (this will for exmaple ensure .gitignore files at the top-level are still checked out). The final git command will set which directory you actually want to be checked out, in our case only the deployment scripts.

This will result in a repository with some files at the top level (like the LICENSE), then a few nested directories and finally only the contents of `deploy/use/production`, exactly what we want.

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

