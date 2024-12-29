# Docker Distribution (Registry)

This is a very simple example of setting up a authenticated v2 Docker Registry.

## Implementation

First let’s clone a Git repository that contains the basic files required to set up a
simple, authenticated Docker registry:

```bash
git clone https://github.com/boxops/basic-registry --config core.autocrlf=input
Cloning into 'basic-registry'…
remote: Counting objects: 10, done.
remote: Compressing objects: 100% (8/8), done.
remote: Total 10 (delta 0), reused 10 (delta 0), pack-reused 0
Unpacking objects: 100% (10/10), done.
```
Once you have the files locally, you can change directories and examine the files that
you just downloaded:
```bash
cd basic-registry
ls
Dockerfile config.yaml.sample registry.crt.sample
README.md htpasswd.sample registry.key.sample
```
The Dockerfile simply takes the upstream registry image from Docker Hub and copies
some local configuration and support files into a new image.
For testing, you can use some of the included sample files, but do not use these in
production.

If your Docker server is available via localhost (127.0.0.1), then you can use these
files unmodified by simply copying each of them like this:
```bash
cp config.yaml.sample config.yaml
cp registry.key.sample registry.key
cp registry.crt.sample registry.crt
cp htpasswd.sample htpasswd
```
If, however, your Docker server is on a remote IP address, then you will need to do a
little additional work.
First, copy config.yaml.sample to config.yaml:
```bash
cp config.yaml.sample config.yaml
```

Then edit config.yaml and replace 127.0.0.1 with the IP address of your Docker
server so that:
```bash
http:
 host: https://127.0.0.1:5000
```
becomes something like this:
```bash
http:
 host: https://172.17.42.10:5000
```
It is easy to create a registry using a fully qualified domain name
(FQDN), like my-registry.example.com, but for this example,
working with IP addresses is easier because no DNS is required.

Next, you need to create an SSL keypair for your registry’s IP address.
One way to do this is with the following OpenSSL command. Note that you will need
to set the IP address in this portion of the command, /CN=172.17.42.10, to match
your Docker server’s IP address:
```bash
openssl req -x509 -nodes -sha256 -newkey rsa:4096 \
 -keyout registry.key -out registry.crt \
 -days 14 -subj '{/CN=172.17.42.10}'
```
Finally, you can either use the example htpasswd file by copying it:
```bash
cp htpasswd.sample htpasswd
```
or you can create your own username and password pair for authentication by using
a command like the following, replacing ${<username>} and ${<password>} with
your preferred values:
```bash
docker container run --rm --entrypoint htpasswd g \
 -Bbn ${<username>} ${<password>} > htpasswd
```
If you look at the directory listing again, it should now look like this:
```bash
ls
Dockerfile config.yaml.sample registry.crt registry.key.sample
README.md htpasswd registry.crt.sample
config.yaml htpasswd.sample registry.key
```
If any of these files are missing, review the previous steps to ensure that you did not
miss one, before moving on.

If everything looks correct, then you should be ready to build and run the registry:
```bash
docker image build -t my-registry .
docker container run --rm -d -p 5000:5000 --name registry my-registry
docker container logs registry
```

If you see errors like “docker: Error response from daemon: Con‐
flict. The container name “/registry” is already in use,” then you
need to either change the preceding container name or remove the
existing container with that name. You can remove the container
by running docker container rm registry.

## Testing the private registry

Now that the registry is running, you can test it. The very first thing that you need
to do is authenticate against it. You will need to make sure that the IP address in
the docker login matches the IP address of your Docker server that is running the
registry.

myuser is the default username, and myuser-pw! is the default
password. If you generated your own htpasswd, then these will be
whatever you choose.
```bash
docker login 127.0.0.1:5000
Username: <registry_username>
Password: <registry_password>
Login Succeeded
```
This registry container has an embedded SSL key and is not using
any external storage, which means that it contains a secret, and
when you delete the running container, all your images will also be
deleted. This is by design.
In production, you will want to have your containers pull secrets
from a secrets management system and use some type of redun‐
dant external storage, like an object store. If you want to keep
your development registry images between containers, you could
add something like --mount type=bind,source=/tmp/registrydata,target=/var/lib/registry to your docker container run
command to store the registry data on the Docker server.

Now, let’s see if you can push the image you just built into your local private registry

In all of these commands, ensure that you use the correct IP
address for your registry.
```bash
docker image tag my-registry 127.0.0.1:5000/my-registry
docker image push 127.0.0.1:5000/my-registry
Using default tag: latest
The push refers to repository [127.0.0.1:5000/my-registry]
f09a0346302c: Pushed
…
4fc242d58285: Pushed
latest: digest: sha256:c374b0a721a12c41d5b298930d11e658fbd37f22dc2a0fac7d6a2…
```
You can then try to pull the same image from your repository:
```bash
docker image pull 127.0.0.1:5000/my-registry
Using default tag: latest
latest: Pulling from my-registry
Digest: sha256:c374b0a721a12c41d5b298930d11e658fbd37f22dc2a0fac7d6a2ecdc0ba5490
Status: Image is up to date for 127.0.0.1:5000/my-registry:latest
127.0.0.1:5000/my-registry:latest
```
It’s worth keeping in mind that both Docker Hub and Docker
Distribution expose an API endpoint that you can query for useful
information. You can find out more information about the API via
the official documentation.

If you have not encountered any errors, then you have a working registry for develop‐
ment and can build on this foundation to create a production registry. At this point,
you may want to stop the registry for the time being. You can easily accomplish this
by running the following:
```bash
docker container stop registry
```
As you become comfortable with Docker Distribution, you may
also want to consider exploring the Cloud Native Computing
Foundation (CNCF) open source project, called Harbor, which
extends the Docker Distribution with a lot of security and
reliability-focused features.
