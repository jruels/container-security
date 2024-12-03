
## Verify seccomp

Run `docker info` and verify the seccomp profile in place.


## Enable User namespacing
Let's enable the namespacing feature of the kernel by doing:

```bash
echo "{\"userns-remap\": \"ubuntu\"}" | sudo tee /etc/docker/daemon.json
```

Then we restart docker:
```
sudo /etc/init.d/docker restart
```

Now run a container, e.g. nginx:

```
docker run -d nginx
```

Look at the userids from the host perspective:
```
ps aux |grep [n]ginx
```

```
ubuntu@ip-172-31-25-93:~$ ps aux |grep [n]ginx
165536   14846  0.0  0.1  10628  5304 ?        Ss   22:10   0:00 nginx: master process nginx -g daemon off;
```

To figure out which id the container is running as check out the uid and gid files. 

`grep <UID from above> /etc/subuid`

```
ubuntu@ip-172-31-25-93:~$ grep 165536 /etc/subuid
ubuntu:165536:65536
```

To figure out the group run 
```
grep <UID from above> /etc/subgid
```

```
ubuntu@ip-172-31-25-93:~$ grep 165536 /etc/subgid
ubuntu:165536:65536
```

You can disable usernamespacing by deleting the daemon file and restarting docker:
```
sudo rm /etc/docker/daemon.json
sudo /etc/init.d/docker restart
```

## Content Trust

#### **Sign Up for a Docker Hub Account**

Docker Hub is a central repository for container images. Signing up allows you to manage and share your images with others. Docker Hub acts as a central repository to store and manage your container images.

- Open a browser and navigate to Docker Hub.
- Click **Sign Up** and follow the prompts to create a free Docker Hub account.
- Verify your email address and ensure you can log in.

#### **Enable Docker Content Trust (DCT)**

Enabling Docker Content Trust ensures that only verified and signed images can be used, enhancing the security of your workflows. By setting the `DOCKER_CONTENT_TRUST` environment variable, you enforce image signing and verification policies, preventing unsigned or tampered images from being used.

- Open a terminal on the lab VM.

- Enable Docker Content Trust by setting the environment variable:

  ```
  export DOCKER_CONTENT_TRUST=1
  ```

- Verify that DCT is enabled:

  ```
  echo $DOCKER_CONTENT_TRUST
  ```

  It should output `1`.

#### **Generate the DCT Signing Key**

Cryptographic signing keys are used to sign Docker images, ensuring their authenticity and preventing tampering. Signing keys provide authenticity and integrity, ensuring the images you push to Docker Hub can be trusted by others.

- Generate a signing key pair for DCT by running:

  ```
  docker trust key generate <your-key-name>
  ```

- When prompted, provide a name for the key (e.g., `docker`) and note the location where the private key is saved.

#### **Build the Docker Image**

The provided Dockerfile packages a simple Go application into a container, creating the base image for the lab. Without signing, it cannot be used with Content Trust enabled. The provided Dockerfile creates a simple Go application and packages it into a container. This forms the base of your containerized application, but the image is not yet signed, which will cause issues when Content Trust is enabled.

- Save the following Dockerfile to a file named `Dockerfile` in the current directory:

  ```
  # syntax=docker/dockerfile:1
  FROM golang:1.23 AS build
  WORKDIR /src
  COPY <<EOF /src/main.go
  package main
  
  import "fmt"
  
  func main() {
    fmt.Println("hello, world")
  }
  EOF
  RUN go build -o /bin/hello ./main.go
  
  FROM scratch
  COPY --from=build /bin/hello /bin/hello
  CMD ["/bin/hello"]
  ```

- Build the image:

  ```
  docker build -t <dockerhub-username>/go-hello .
  ```

#### **Try Running the Container**

Attempting to run the unsigned image highlights Docker Content Trust's enforcement of signature requirements, resulting in a `401 Unauthorized` error. Since Docker Content Trust is enabled, the unsigned image fails to run with a `401 Unauthorized` error. This happens because the image lacks a valid signature, which Content Trust requires to ensure the image’s authenticity.

- Attempt to run the container:

  ```
  docker run <dockerhub-username>/go-hello
  ```

- Observe that it fails with the error:

  ```
  docker: you are not authorized to perform this operation: server returned 401.
  ```

#### **Log In to Docker CLI**

Logging into Docker Hub through the CLI is required to push images and manage signing keys. Logging in is necessary to push the Docker image to your registry and to generate signing keys for Docker Content Trust during the push process.

- Log in to Docker CLI with your Docker Hub credentials:

  ```
  docker login -u <dockerhub-username>
  ```

#### **Push the Image to Docker Hub**

Pushing the image to Docker Hub stores it in your registry and triggers the signing process when Content Trust is enabled. When Content Trust is enabled, Docker prompts you to generate signing keys during this step. These keys ensure that your image is signed and can be verified for authenticity.

- Push the image to Docker Hub:

  ```
  docker push <dockerhub-username>/go-hello
  ```

- Docker will prompt you to create signing keys. Follow the prompts to generate and set up the DCT keys.

#### **Sign the Image**

Signing the image secures it against tampering and allows others to verify its authenticity. Signed images provide assurance that the image has not been tampered with and is safe to use.

- After pushing the image, sign it using Docker Content Trust:

  ```
  docker trust sign <dockerhub-username>/go-hello
  ```

#### **Run the Signed Container**

Running the signed image verifies its integrity and demonstrates the successful enforcement of Docker Content Trust. After signing the image and pulling it from Docker Hub, the container runs without issues, demonstrating the effectiveness of Docker Content Trust in verifying image authenticity.

- Pull the signed image from Docker Hub:

  ```
  docker pull <dockerhub-username>/go-hello
  ```

- Run the container again:

  ```
  docker run <dockerhub-username>/go-hello
  ```

- Observe that the container runs successfully, and you see:

  ```
  hello, world
  ```

#### **Disable Docker Content Trust**

Disabling Docker Content Trust allows unsigned images to be used, which is suitable for development but not recommended for production. This allows you to use unsigned images, which can be useful for development or testing purposes, but it’s not recommended for production environments.

- Disable DCT by unsetting the environment variable:

  ```
  unset DOCKER_CONTENT_TRUST
  ```

- Verify that DCT is disabled:

  ```
  echo $DOCKER_CONTENT_TRUST
  ```

  It should output nothing.


## Running as non-root

You may recall that you can build a Go program as a static binary and, using multi-stage builds, insert it on the scratch image like so:

```
FROM golang:1.9-alpine

COPY main.go /go/src/github.com/chrishiestand/docker-go-hello-world/

RUN cd /go/src/github.com/chrishiestand/docker-go-hello-world && \
    go get && \
    CGO_ENABLED=0 GOOS=linux go build -a -o /go/bin/hello main.go

FROM scratch
CMD ["/app"]
COPY --from=0 /go/bin/hello /app
```

Now to improve security further, let's make this application not run as root. You will need to use the `USER` dockerfile command. You will also need an `/etc/passwd` file in your new image. You can get one from a number of places, or create one, but it's probably simplest now to copy the file from the golang build container. To find out what unprivileged users are in that file you can `docker run` the golang build image and `cat /etc/passwd`.


## Linux Capabilities

Run this command to see the current linux capabilities:
```
sudo /sbin/capsh --print
```

Now start this container that immediately installs the capsh program:
```
docker run -it debian /bin/sh -c "apt-get update && apt-get install -qq libcap2-bin && bash"
```

And then from inside the container run:
```
/sbin/capsh --print
```

Note the difference between the output on the host and inside the container. You can also try starting containers with dropped capabilities:


```
docker run --cap-drop=net_raw  -it debian /bin/sh -c "apt-get update && apt-get install -qq libcap2-bin && bash"
```
And once again run:
```
/sbin/capsh --print
```

Note the dropped capability is missing from the output inside the container.
