---
layout: blog/post
title:      Get the natted port from inside a Docker container
date:       2016-07-20
author: totomz
img_folder: '/images/posts/old'  
img_cover: '627.jpg'
excerpt_separator: <!--more-->
---

This is a simple workaround to get virtually any information of the Docker host from the inside of a running container.  I am using it to get the value of the natted ports that are exposed to the outside by the container
<!--more-->

> If you need to get some host informations from inside a container, you are **doing something wrong**. Probably there is a better solution that that, *a container should not know that it is a container*....

A common solution is to expose the socket of the Docker Remote API to the container mounting it with: `-v /var/run/docker.sock:/var/run/docker.sock`

This i s a**major security issue**, as stated on [this StackOverflow answer](http://stackoverflow.com/a/33183227/571043):

1. All host containers will be accessible to container, so it can stop them, delete, **run any commands as any user inside top-level Docker containers**.
2. All created containers are created in a top-level Docker.
4. Of course, you should understand that if container has access to host's docker, it has privileged access to host system. Depending on container and system (AppArmor) configuration, it may be less or more dangerous
5. Other warnings here https://www.lvh.io/posts/dont-expose-the-docker-socket-not-even-to-a-container.html

**This  solution** gives you enough freedom to expose *what you need* (eg: the list of natted port), and is **pretty easy**:

1. On the host, start a simple socket server that expose *only* the docker port list (it can be easily wrapped in a background process that starts with the system, and can be firewalled)

`ncat -k -l -p 5555 -c 'read i && echo $(docker port $i | paste -d, -s)'`

2.  Start the container with the docker host ip in an environment variable

`docker run --rm -ti -p 8000 -p 8742 -e "pippo=$(ip route get 8.8.8.8 | awk 'NR==1 {print $NF}')"  ubuntu /bin/bash`

3. To get the list of natted port from inside a container

`echo $(hostname) | nc $pippo 5555`


It can be used from everywhere inside a container, eg: from java

{% highlight java %}
```
try(Socket echoSocket = new Socket(System.getenv("pippo"), 5555)) {

            PrintWriter out = new PrintWriter(echoSocket.getOutputStream(), true);
            BufferedReader in = new BufferedReader(new InputStreamReader(echoSocket.getInputStream()));

            // Container ID
            out.println("2bdb596c16be");

            // One line with all the infos
            System.out.println(in.readLine());

            System.out.println("Done");

        }
```
{% endhighlight %}

Done!

# Run on OSX
*EDIT:* This post has been edited to include the comments from [degree](https://github.com/docker/docker/issues/3778#issuecomment-233987356)

To run the workaround on a mac

{% highlight bash %}
```
$ brew tap brona/iproute2mac
$ brew install iproute2mac
$ ncat -k -l -p 5555 -c 'read i && echo $(docker port $i | paste -sd, -)' &
$ docker run --rm -it -e "pippo=$(ip route get 8.8.8.8 | awk 'NR==1 { print $NF }')" -p 8123 -p 8512 ubuntu /bin/bash
```
{% endhighlight %}

In the container

{% highlight bash %}
$ apt-get update; apt-get install -y netcat
$ echo $HOSTNAME | nc $pippo 5555
8123/tcp -> 0.0.0.0:32774,8512/tcp -> 0.0.0.0:32773
$ echo $HOSTNAME 8123 | nc $pippo 5555
0.0.0.0:32774
$ echo $HOSTNAME 8512 | nc $pippo 5555
0.0.0.0:32773
{% endhighlight %}

Result (respectively per request):

```
8123/tcp -> 0.0.0.0:32774,8512/tcp -> 0.0.0.0:32773
0.0.0.0:32774
0.0.0.0:32773
```

Note:

1. 8123 and 8512 are just for example
2. paste command is reformatted
3. in container $HOSTNAME is used instead of version with parenthesis
4. if you specify also port then you get spesific result
