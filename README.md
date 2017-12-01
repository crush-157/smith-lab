# Smith Hands On Exercise

## Introduction
[Smith](https://github.com/oracle/smith) is a command line tool for building [microcontainers](https://blogs.oracle.com/developers/the-microcontainer-manifesto).

The aims of this exercise are to walk you+ through:
1. Installing Smith
2. Building a "hello world" microcontainer image from scratch
3. Shrinking an existing container

## Pre - requisites
- Linux machine (physical or virtual)
- Editor (vim _obviously_)
- Docker installed
- git installed
- a dockerhub account

## Installing Smith
1. Navigate to [https://github.com/oracle/smith](https://github.com/oracle/smith)
2. Clone the repo to your machine
3. Install build dependencies (as described in the Smith Readme).
4. If you're using Debian (or a Debian derived distro), make sure you follow the additional steps for installing mock _(or bad things **will** happen)_
5. Run `sudo make install`
6. Run `smith --version`

If the response is something like `smith version 1.1.2.22.bc4d01a (built from sha bc4d01a)` then you're in good shape.

## Building a "hello world" microcontainer from scratch
1. Create a new directory to build your container in.
2. Set environment variables for your DOCKER_ID and DOCKER_PWD (as used in the examples below).
3. Create a smith.yaml file:
```
package: coreutils
paths:
- /usr/bin/cat
cmd:
- /usr/bin/cat
- /read/data
```

This is to build a microcontainer image from scratch.

Stepping through the file:

`package: coreutils` defines as the source.
```
paths:
- /usr/bin/cat
```
Adds `/usr/bin/cat` to the microcontainer image.
```
cmd:
- /usr/bin/cat
- /read/data
```
Is a command to `cat` the file `/read/data`.

Create a subdirectory `rootfs` from your working directory.  Any files or directories placed in here will be appear under root in the image `/`.

So to add '/read/data' to the image, create a subdirectory `read` of `rootfs`.  Then under read, create a file `data` and put the string you'd like to display (e.g. 'Hello World!') in it.

So you should have a structure like this:
```
[ewan@empiricist cat]$ ls -R
.:
rootfs  smith.yaml

./rootfs:
read

./rootfs/read:
data
[ewan@empiricist cat]$ cat rootfs/read/data
Hello World!
```

Now switch back to your working directory and run `smith -i hello.tgz`.

Once that has finished, you should have a `hello.tgz` file in your working directory.

To get Docker to run this, we need to upload it to a Docker repo (e.g. dockerhub) to turn it into Docker format (_standards are wonderful when they're implemented_):

`smith upload -r https://$DOCKER_ID:$DOCKER_PWD@registry-1.docker.io/$DOCKER_ID/hello-smith -i hello.tgz`

Then you should be able to run it:

`docker run --rm $DOCKER_ID/hello-smith`

For example:
```
ewan@starbug:~/projects/smith-examples/cat$ docker run --rm $DOCKER_ID/hello-smith
Unable to find image 'crush157/hello-smith:latest' locally
latest: Pulling from crush157/hello-smith
ca43e03ec88e: Pull complete
Digest: sha256:eb06aa9738691a7b53f2ee420600be18f473c8fccc40a53a13e683fd40c7f2eb
Status: Downloaded newer image for crush157/hello-smith:latest
Hello World!
```
**Coffee time :-)**

## Shrinking an existing image

The image we're going to shrink contains a rudimentary "ticketing system" called [Dogsbody](https://github.com/crush-157/dogsbody).  It exists only for the purpose of exercises and labs like this one.

If you're interested the source code is on [GitHub](https://github.com/crush-157/dogsbody).

There is already a container image on Docker Hub [crush157/dogsbody](https://hub.docker.com/r/crush157/dogsbody/).

If you prefer to shrink a different image, then you can just use this as an example.  If you want another example you can also look at [How To Build a Tiny Httpd Container](https://hackernoon.com/how-to-build-a-tiny-httpd-container-ae622c37db39).

The Dockerfile for the base Dogsbody image is as follows:

```
FROM ruby

RUN apt-get update

# Install app
RUN mkdir /app
WORKDIR /app
ADD ./dogsbody.tgz .
RUN bundle install
CMD ./run.sh
```

If you pull this down to your machine you'll see that it has a size of 754MB.

Now want to shrink this.

1.  Create another working directory (it's just tidier that way), and cd into it.
2.  Download the image in OCI format:

```smith download -r https://$DOCKER_ID:$DOCKER_PWD@registry-1.docker.io/crush157/dogsbody -i dogsbody-image.tgz```
3.  Create a bare bones smith.yaml file:
```
type: oci
package: dogsbody-image.tgz
paths:
- /app/
```
4.  Run smith ```smith -i dogsbody.tar.gz```.  This will create an image with not much in it, but it will also unpack the OCI image for us to have a look around.
5.  If you look under `/tmp` you will see a directory with a name something like `smith-unpack-1000` (the last part will be your UID so may be different).  In there are the contents of the OCI container.  Let's have a look around for ruby and sinatra:
```
ewan@starbug:/tmp/smith-unpack-1000$ find ./ -name ruby
./usr/local/bin/ruby
./usr/local/lib/ruby
./usr/local/include/ruby-2.4.0/x86_64-linux/ruby
./usr/local/include/ruby-2.4.0/ruby
ewan@starbug:/tmp/smith-unpack-1000$ find ./ -name sinatra
./usr/local/bundle/gems/sinatra-2.0.0/lib/sinatra
./usr/local/bundle/gems/mustermann-1.0.1/lib/mustermann/sinatra
./root/.bundle/cache/compact_index/rubygems.org.443.29b0360b937aa4d161703e6160654e47/info/sinatra
```
6.  So lets add some likely looking paths to our smith.yaml, and the command to run the app:
```
type: oci
package: dogsbody-image.tgz
paths:
- /app/
- /usr/local/bin/
- /usr/local/lib/
- /usr/local/include/
- /usr/local/bundle/
cmd:
  - ruby
  - app.rb
  - '-e production'
```
7.  Let's upload it to docker hub, to get an image in Docker format:

`smith upload -r https://$DOCKER_ID:$DOCKER_PWD@registry-1.docker.io/$DOCKER_ID/smith-dogsbody -i dogsbody.tar.gz`

8.  Now let's try and run it.  The first thing we want to do is run `rake db:migrate` which checks the db schema and updates it if necessary.  Replace the MYSQLCS_* environment variables in the example command below with the appropriate values for your database and then run it:
```
ewan@starbug:~/projects/smith-examples/dogsbody$ docker run -it --rm \
>   --name dogsbody \
>   --read-only \
>   --env MYSQLCS_CONNECT_STRING="172.17.0.1:3306/dogsbody" \
>   --env MYSQLCS_USER_PASSWORD="Welcome_1" \
>   --env MYSQLCS_USER_NAME="root" \
>   $DOCKER_ID/smith-dogsbody rake db:migrate
Migrating to latest
```
9.  If it returned "Migrating to latest" you should be good to run the service.  Replace the MYSQLCS_* environment variables in the example command below with the appropriate values for your database and then run it:
```
ewan@starbug:~/projects/smith-examples/dogsbody$ docker run -d --rm \
>   --name dogsbody \
>   --read-only \
>   --env MYSQLCS_CONNECT_STRING="172.17.0.1:3306/dogsbody" \
>   --env MYSQLCS_USER_PASSWORD="Welcome_1" \
>   --env MYSQLCS_USER_NAME="root" \
>   --env PATH="/usr/local/bin" \
>   --publish 22222:4567/tcp \
>   $DOCKER_ID/smith-dogsbody
5c7931542a7c5b41ab517b451dac37c8224bcca1fb8d1fbd91989c9a887727dc
```
10.  Check that it's running:
```
ewan@starbug:~/projects/smith-examples/dogsbody$ docker ps
CONTAINER ID        IMAGE                       COMMAND                  CREATED             STATUS                   PORTS                               NAMES
5c7931542a7c        crush157/smith-dogsbody     "ruby app.rb '-e p..."   36 seconds ago      Up 35 seconds            0.0.0.0:22222->4567/tcp             dogsbody
640f14c778ad        mysql/mysql-server:latest   "/entrypoint.sh my..."   12 days ago         Up 5 minutes (healthy)   0.0.0.0:3306->3306/tcp, 33060/tcp   mysql
ewan@starbug:~/projects/smith-examples/dogsbody$ curl -v localhost:22222/users
*   Trying ::1...
* TCP_NODELAY set
* Connected to localhost (::1) port 22222 (#0)
> GET /users HTTP/1.1
> Host: localhost:22222
> User-Agent: curl/7.52.1
> Accept: */*
>
< HTTP/1.1 200 OK
< Content-Type: application/json
< Content-Length: 2
< X-Content-Type-Options: nosniff
< Connection: keep-alive
< Server: thin
<
* Curl_http_done: called premature == 0
* Connection #0 to host localhost left intact
[]
```
200 is what we want.
11.  Let's check the image size:
```
ewan@starbug:~/projects/smith-examples/dogsbody$ docker images
REPOSITORY                 TAG                 IMAGE ID            CREATED             SIZE
crush157/smith-dogsbody    latest              5bbfc75d826b        12 minutes ago      112MB
crush157/hello-smith       latest              a430ff69520c        2 hours ago         2.3MB
ruby                       2.4.2-alpine3.6     8647c16e6bb0        8 days ago          72MB
crush157/smith-httpd       latest              9d1ba46c4fb0        11 days ago         4.76MB
crush157/squash-dogsbody   latest              a8e0ddf918b5        12 days ago         754MB
crush157/dogsbody          latest              18e67fa6c47e        12 days ago         754MB
jruby                      latest              300c281a1aca        13 days ago         594MB
ruby                       latest              c7715c1eb8fe        2 weeks ago         687MB
httpd                      latest              74ad7f48867f        2 weeks ago         177MB
debian                     latest              6d83de432e98        2 weeks ago         100MB
alpine                     latest              053cde6e8953        2 weeks ago         3.97MB
mysql/mysql-server         latest              a3ee341faefb        5 weeks ago         246MB
```
And we see that we've gone from 754MB down to 112MB!

With a little extra work, you can even knock of another 2MB!  The smith.yaml for that is:
```
type: oci
package: dogsbody-image.tgz
paths:
- /app/
- /usr/local/bin/ruby
- /usr/local/bin/rake
- /usr/local/lib/libruby.so
- /usr/local/lib/ruby
- /usr/local/bundle
cmd:
- ruby
- app.rb
- '-e production'
```
