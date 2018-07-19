# Nginx for Docker Swarm Services

This is a stock Nginx container with a very simple configuration file that simply do a reverse proxy of any request to a Docker Swarm service. The service-name is part of the request-url.

Say you have Docker Swarm service named *voldemort*, running on an overlay network named *api-internal*. If you start a service with this nginx image, using the same network, and give it a reachable port, it will forward your request to port 8080 on that service.

# Background

I wrote a Docker Swarm service that will be deployed in large numbers, using unique names (thousands of service-names for the same Docker image, just with a some individual configuration). When I looked for a simple way to expose them to the micro-services that will use them, (no containers, no Swarm) I found nothing. I found projects that tried to configure nginx, [HAProxy](http://www.haproxy.org/) and other load balancers via REST, or automatically when services were added or removed. But nothing - simple - nothing that just works out of the box. So I discussed the problem with a friend, and after some feedback from him, and some more research, I came up with this solution.

Basically, we just suck the service-name (which is a dns entry in the overlay network) out of the url, assign it to a variable, and do a reverse proxy operation to whatever is in that variable.

```
    location ~ ^/proxy/(?<site>[^/]+)/? {
        rewrite ^.*\/proxy\/(?<site>[^\/]+)\/?(.*) /$2 break;
        proxy_pass http://$site:8080;
    }
```

In this case the service is running on port 8080, so we forward to that port.

# A simple example

In this example I run commands on a node in a Swarm.
```sh
$ docker network create  --driver overlay api-internal
s8rzqedokarlg5hywv9isyku1

$ docker service create --name voldemort -e SERVICE_NAME=Voldemort --network api-internal --replicas 4 jgaafromnorth/rest-diag-svc
7og8izi0cu1k276vxeqn24bfw
overall progress: 4 out of 4 tasks
1/4: running   [==================================================>]
2/4: running   [==================================================>]
3/4: running   [==================================================>]
4/4: running   [==================================================>]

$ docker service create --name proxy -p 80:80 --network api-internal --replicas 2 jgaafromnorth/nginx-swarm-proxy

overall progress: 2 out of 2 tasks
1/2: running   [==================================================>]
2/2: running   [==================================================>]
verify: Service converged

$ docker service ls
ID                  NAME                MODE                REPLICAS            IMAGE                                    PORTS
oz71x1igkwkz        proxy               replicated          2/2                 jgaafromnorth/nginx-swarm-proxy:latest   *:80->80/tcp
7og8izi0cu1k        voldemort           replicated          4/4                 jgaafromnorth/rest-diag-svc:latest

```

And now, let's send a query to voldemort, via the proxy

```sh
$ curl "http://127.0.0.1/proxy/voldemort/search/adversaries?first_name=Harry&last_name=Potter"
```

The response in my case, using an image for *voldemort* that just returns diagnostics data:

```json
{
    "service": "Voldemort",
    "host": "545e96dc95ad",
    "request": "/search/adversaries?first_name=Harry&last_name=Potter",
    "method": "GET",
    "headers": {
        "x-forwarded-for": "10.255.0.2",
        "host": "127.0.0.1/proxy/voldemort",
        "x-forwarded-proto": "http",
        "connection": "close",
        "user-agent": "curl/7.54.1",
        "accept": "*/*"
    },
    "from": "::ffff:10.0.0.58"
}
```

# Prebuilt image

An image built from this project's Dockerfile is available from [Docker Hub](https://hub.docker.com/r/jgaafromnorth/nginx-swarm-proxy/)

```sh
$ docker pull jgaafromnorth/nginx-swarm-proxy
```

# A word of Caution!

My use-case is for services and consumers that run inside a secure environment. The proxies are not receiving any requests from Internet, or even from corporate networks, except for the servers that will consume them. If you deploy this thing in an insecure environment, make sure to harden it so you don't open up an anonymous proxy-server to the Internet (or to your internal network).

