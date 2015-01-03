---
layout: post
title: "Using Redis as Tomcat Session Store with Docker"
description: ""
category: "Continuous Integration"
tags: ["Docker", "Redis", "Tomcat", "Session" ]
summary: "How to use Redis as Session Store for Tomcat to make the session
available to all Tomcat instances in a cluster."
image: "/img/posts/docker.png"
---
{% include JB/setup %}

This is the first part of a series of posts on how to create a scalable Java
Application with Docker. TDD and Integration Tests helps us a lot to have some trust in our source
code. Amongst other things, Docker helps us to have more trust in our
applications as a whole, by running whole applications in a production-like
cluster setup on a single host. In this initial post I will prepare a Tomcat
Docker Image which uses Redis as a Session Store, which we can use in later
posts to achive high availability for our user sessions.


Start a Redis Server
--------------------

To start a Redis server simply start the official [Redis Docker
Image](https://registry.hub.docker.com/_/redis/).

{% highlight bash %}
docker run -d --name redis-session-store redis
{% endhighlight %}

Now verify that the Redis server is working properly.

{% highlight bash %}
docker run -it --link redis-session-store:redis --rm redis sh -c 'exec redis-cli -h "$REDIS_PORT_6379_TCP_ADDR" -p "$REDIS_PORT_6379_TCP_PORT"'
172.17.0.5:6379> INFO keyspace
# Keyspace
{% endhighlight %}

You should now be logged in to your Redis Server. When typing `INFO keyspace`,
no surprise, there is no keyspace yet.

Prepare the Tomcat Image
----------------------

To use Redis as the Session Store, a Session Manager Implementation is
required. Luckily a project called
[tomcat-redis-session-manager](https://github.com/jcoleman/tomcat-redis-session-manager)
exists, which provides an implementation. The latest official release of the
project is quite old, so I forked it and added a compiled jar of the master
branch on
[Github](https://github.com/rmohr/tomcat-redis-session-manager/releases).  As
Tomcat base image I have used
[tutum/tomcat:7.0](https://registry.hub.docker.com/u/tutum/tomcat/). Next the
Session Manager and its dependencies needed to be placed in the image:

{% highlight bash %}
RUN \
    cd /tomcat/lib && \
    wget -q  https://github.com/rmohr/tomcat-redis-session-manager/releases/download/2.0-tomcat-7/tomcat-redis-session-manager-2.0.0.jar && \
    wget -q http://central.maven.org/maven2/redis/clients/jedis/2.5.2/jedis-2.5.2.jar && \
    wget -q http://central.maven.org/maven2/org/apache/commons/commons-pool2/2.2/commons-pool2-2.2.jar
{% endhighlight %}

For convenience the two environment variables `REDIS_HOST` and `REDIS_PORT`
were added to the Dockerfile. They will be evalueted on runtime. Default values
are `redis` and `6379`, which are sufficient when linking the containers
correctly.

{% highlight bash %}
ENV REDIS_HOST redis # Redis server host name
ENV REDIS_PORT 6379 # Redis server port
{% endhighlight %}

Start the Tomcat Image
----------------------

Luckily I have created a [rmohr/tomcat-redis Trusted Build Image with the
Session Manager
built-in](https://registry.hub.docker.com/u/rmohr/tomcat-redis/).
You can see the [Dockerfile on
Github](https://github.com/rmohr/docker-tomcat-redis/). To start the Image and
link it against the Redis Image, simply run

{% highlight bash %}
docker run --link redis-session-store:redis -p 8080:8080 rmohr/tomcat-redis:7.0
=> Creating and admin user with a random password in Tomcat
=> Done!
========================================================================
You can now configure to this Tomcat server using:

    admin:XPkpXKB8yuzS

========================================================================
{% endhighlight %}

Got to `http://localhost:8080/manager/html` and log in with the credentials you
see on the Tomcat container output. Finally login to the Redis Server again and
do the following:

{% highlight bash %}
$ docker run -it --link redis:redis --rm redis sh -c 'exec redis-cli -h "$REDIS_PORT_6379_TCP_ADDR" -p "$REDIS_PORT_6379_TCP_PORT"'
172.17.0.5:6379> INFO keyspace
# Keyspace
db0:keys=1,expires=1,avg_ttl=1696125
172.17.0.5:6379> KEYS '*'
1) "C83B106DBE9E5CB28EEC7E0B463628E1"
172.17.0.5:6379> GET "C83B106DBE9E5CB28EEC7E0B463628E1"
"\xac\xed\x00\x05sr\x00Dcom.orangefunction.tomcat.redissessions.SessionSerializationMetadataB\xd9\xd9\xf7v\xa2\xdbL\x03\x00\x01[\x00\x15sessionAttributesHasht\x00\x02[Bxpw\x14\x00\x00\x00\x10tcon?\xc1\xaa\x827\x99\x0by\x93=|\xcdxsr\x00\x0ejava.lang.Long;\x8b\xe4\x90\xcc\x8f#\xdf\x02\x00\x01J\x00\x05valuexr\x00\x10java.lang.Number\x86\xac\x95\x1d\x0b\x94\xe0\x8b\x02\x00\x00xp\x00\x00\x01J\xab{h\xb4sq\x00~\x00\x03\x00\x00\x01J\xab{h\xb4sr\x00\x11java.lang.Integer\x12\xe2\xa0\xa4\xf7\x81\x878\x02\x00\x01I\x00\x05valuexq\x00~\x00\x04\x00\x1bw@sr\x00\x11java.lang.Boolean\xcd r\x80\xd5\x9c\xfa\xee\x02\x00\x01Z\x00\x05valuexp\x00sq\x00~\x00\t\x01sq\x00~\x00\x03\x00\x00\x01J\xab}\x91\xa3t\x00 C83B106DBE9E5CB28EEC7E0B463628E1sq\x00~\x00\a\x00\x00\x00\x01t\x00&org.apache.catalina.filters.CSRF_NONCEsr\x009org.apache.catalina.filters.CsrfPreventionFilter$LruCache\x00\x00\x00\x00\x00\x00\x00\x01\x02\x00\x01L\x00\x05cachet\x00\x0fLjava/util/Map;xpsr\x00;org.apache.catalina.filters.CsrfPreventionFilter$LruCache$1\x00\x00\x00\x00\x00\x00\x00\x01\x02\x00\x02I\x00\rval$cacheSizeL\x00\x06this$0t\x00;Lorg/apache/catalina/filters/CsrfPreventionFilter$LruCache;xr\x00\x17java.util.LinkedHashMap4\xc0N\\\x10l\xc0\xfb\x02\x00\x01Z\x00\x0baccessOrderxr\x00\x11java.util.HashMap\x05\a\xda\xc1\xc3\x16`\xd1\x03\x00\x02F\x00\nloadFactorI\x00\tthresholdxp?@\x00\x00\x00\x00\x00\x00w\b\x00\x00\x00\x01\x00\x00\x00\x01t\x00 AD33B7B7EC4106A74733738E12E8FAD2px\x00\x00\x00\x00\x05q\x00~\x00\x12w\b\x00\x00\x01J\xab{h\xb4"
{% endhighlight %}

Tomcat successfully stored the Session in Redis.

A minimal [Fig](http://www.fig.sh/) setup with persistence for Redis enabled
might look like this:


{% highlight yaml %}
redis:
    image: redis
    volumes:
        - ./data:/data

tomcat:
    image: rmohr/tomcat-redis:7.0
    ports:
        - 8080:8080 
    links:
        - redis:redis
{% endhighlight %}

In the next post I will show you, how to configure Redis and the Tomcat Image
to use Redis Sentinel, for achieving automatic failover.


