# Dockerfiles

The main objective of the exercise is to add a Dockerfile to a Python App. So we have an application and we are asked to do the following:
 * we need to create a Dockerfile that runs our python application and expose on a well defined port, let's say the 9000.
   On that port the app will return a JSON payload shaped as
   ```json
   {
       env: <environment>
   }
   ```
   where `<environment>` is a string defined through the environment variable `ENVIRONMENT` that we can control through Docker itself
 * The Docker will be run setting the `ENVIRONMENT` variable, for example, to `"production"`.


### The App
First of all, let's move on a different root directory called `flask`:

```bash
mkdir -p flask
cd flask
```

A simple python3 API that only responds at `/`. It returns the value
of the `ENVIRONMENT` environment var as JSON. Save the following code in a file named `app.py`:

```python
from flask import Flask
import os
import sys

app = Flask(__name__)

@app.route("/")
def index():
    env = os.getenv("ENVIRONMENT", None)
    return {
        "env": env
    }

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print("Needs port number as a commandline argument.")
        sys.exit(1)
    port = int(sys.argv[1])
    app.run(host='0.0.0.0', port=port)
```

And also, let's add a simple entrypoint that looks like `startup.sh`:

```bash
#!/bin/bash

python app.py ${PORT}
```

### The  Dependencies

Our App has some dependences, so we need to take care of them, here below the list of the software dependences:

```
Click==7.0
Flask==1.1.1
itsdangerous==1.1.0
Jinja2==2.11.3
MarkupSafe==1.1.1
Werkzeug==0.16.0
```

##### Tip 
>    We will Use `pip` to install the requirements listed in `requirements.txt`:
>    * `pip install -r requirements.txt`

### Running the server

Of course we want our App be up&running. The server requires one command line argument: the port that it should listen on. 
To run the app we need just something like 

`python app.py <PORT>`

### Compose the Dockerfile

Let's now build the *Dockerfile* in order to pack our app. 

A *Dockerfile* is nothing but a text file named `Dockerfile` with one command per line.
Each command features arguments that define its behavior.

For example,

```Dockerfile
## FROM defines the image to start from. It will be pulled automatically if not available locally.
FROM python:3.7-slim-buster

## RUN executes a shell command, here we install curl
RUN apt-get update && apt-get install -y curl

## COPY imports files from the host to the container file system
COPY requirements.txt /tmp/

RUN pip install -r /tmp/requirements.txt

## ENV sets environment variables
ENV PORT="3000"
ENV ENVIRONMENT="prod" 

## EXPOSE defines ports intended to be exposed, note how we use env variables defined above
EXPOSE ${PORT}

## It is a good idea to create dedicated users to run the containers instead of running as root
RUN useradd --create-home pythonappuser
## WORKDIR define the working directory where docker run will land to
WORKDIR /home/pythonappuser

## We import the the Flask application scripts
COPY startup.sh .
COPY app.py .

## We make them executable and owned by the new user
RUN chmod +x startup.sh && chmod +x app.py && chown -R pythonappuser:pythonappuser .

## USER selects the default user docker run will land to
USER pythonappuser

## Entrypoint defines the script executed by docker run. Arguments to docker run are passed
## to the entry point.
ENTRYPOINT ["./startup.sh"]
```

Now we can try to build it: 

```bash
docker build -t mycontainerizedapp .
```

### Playing with our Dockerized - App

First of all let's check it is doing what is expected. So we should run the container and check that the Python App is actually running. So

```bash
docker run -d mycontainerizedapp
```

now we can get the ID of the container ( you know how to do that now ) and then we can get a bash in to our container: 

```bash
docker exec -ti 9ea45699221b bash
```

Now if it is correctly running it should tell us something if we ask someting on its port 3000 ( remmember what we set in our Dockerfile above )

```bash
pythonappuser@9ea45699221b:~$ curl localhost:3000
{"env":"prod"}
pythonappuser@9ea45699221b:~$ 
```
here we go!! it works nicely. 

#### Exposing the port
Very good, let's do a step further. We know that we can map the exposed port to the HOST.. so let's do it 

```bash
docker run -d -p 3000:3000 mycontainerizedapp 
```
if it is all working now I shouldn't need to enter the container to check the status of the App on port 3000.. let's try

```bash
curl localhost:3000
{"env":"prod"}
```
It works. So all is ok. 
!!! tip
    Remember that the `EXPOSE` instruction does not actually publish the port. It functions as a type of documentation between the person who builds the image and the person who runs the container, about which ports are intended to be published

#### Changing the environment variables
Final step now is about the env variable. Can we re-define it when we startup the continer ? yes we can, remember the -e 

```bash
 docker run -d -p 3000:3000 -e ENVIRONMENT=ciccio mycontainerizedapp 
```

If it worked correctly we should get something different wrt the previous test by querying the port 3000.. let's try: 

```bash
curl localhost:3000
{"env":"ciccio"}
```
indeed! All good! 

#### Exercise
You may want to try to define the content of the web app response through a file in the host file system, and pass it to the containerized application by binding it to the docker volume.



