---
title: "Migrating Convox Gen 1 applications to Gen 2"
date: 2017-12-14T14:49:35-05:00
draft: true
---

For those who are unfamiliar with Convox (https://convox.com/), it's basically basically Heroku for your AWS account. It takes all those
amazing aws services and exposes them via an easy to use CLI tool. I highly recommend it for anyone looking for an easy to use
deployment strategy for applications built to use Docker.

Recently Convox went live with what is being called their "Generation 2" application stack. While
gen 2 offers many improvements over gen 1, unfortunately there is no inline-upgrade path.

One of the major differences, and the reason I personally transitioned my applications to gen 2,  is the replacement 
of per-service Elastic Load Balancers with a single, or in some cases, two Application load balancers for an entire rack.
Individual ELBs cost ~\$18.00/month while ALBs cost ~\$12.00/month. For anyone running many services the cost of many ELBs adds up quickly. 
Between multiple environments and this new micro-service we all live in, you might end up with a significant amount of ELBSs.

# Preparing your rack
The first thing you need to do is upgrade the rack itself. It's good to note that you're upgrading the rack to support gen 2 applications. Your existing 
applications will not break and for the immediate future you are still able to create new gen 1 applications even after upgrading.

To upgrade your rack all you have to run is:
{{< highlight bash >}}
convox rack update
{{< / highlight >}}

If you're upgrading from a older version, be patient, it could take up to 15 minutes for the rack api to come back online, 
and during this time it will be an 'unreachable' state.

Once finished, if you have any "internal" services within your applications, you'll need to set the Internal flag to true on the rack in order to run internal gen 2 services.
{{< highlight bash >}}
convox rack params set Internal=Yes
{{< / highlight >}}

# Creating Your Gen 2 Application
You can fairly easily "clone" an existing application's env vars by running:
{{< highlight bash >}}
convox apps create <TEMP_APP_NAME> --generation 2 --wait
convox env -a <APP_NAME> | convox env set -a <TEMP_APP_NAME> --wait
{{< / highlight >}}

# Preparing your application
Generation 1 applications supported a subset of a valid `docker-compose.yml` file. However for gen 2 applications, there is now a dedicated `convox.yml` file that you use to 
define your application's infrastructure. While very similar to your `docker-compose.yml` file there are some slight differences. You can find a full example and docs for `convox.yml` here: https://convox.com/docs/gen2/convox-yml/

A very simple django application might look something like this:
{{< highlight yml >}}
services:
  django:
    build: .
    command: /gunicorn.sh
    port: 5000
    health: /health/
    internal: true
    domain: ${HOST}

  nginx:
    build: compose/nginx/
    port: 80
    health: /vw-status
    links:
      - django

  celeryworker:
    build: .
    command: /worker.sh

  celerybeat:
    build: .
    command: /beat.sh
{{< / highlight >}}

#### Important Notes:
##### On domain
If you want to serve your application at a custom domain, you must use the `domain`. attribute on the service. The `${HOST}` value can be any valid domain name, but in this example we use 
and environment variable so that we can use different domains when the application is deployed to multiple racks.

It's also important that you have an email address setup at `admin@DOMAIN.TLD`. Keep it open during the first deployment using your domain. Convox will automatically provision an SSL certificate for the domain
and you must open the email and validate the cert before the application will deploy successfully.

Also, if you wish to use env vars for your domain, I would recommend adding the env vars to the original gen 1 application before creating the gen 2 application, just so when you clone the env vars you don't have to
worry about them not existing.

##### On health
Health checks are a requirement for any application that exposes a port. This is a requirement of ALBs and without a healthy check the application will never stabilize. This url must return specifically a 
200 status to an anonymous request. Even statuses such as 301 redirects will be considered unhealthy. This one bit me a few times, and I recommended ensuring your application has a valid health path
before trying to do your first deployment.


# Deploying your new gen 2 application.
Once you think you have everything setup you can run:
{{< highlight bash >}}
convox deploy -a  <TEMP_APP_NAME>
{{< / highlight >}}
to deploy your new gen 2 application.

Once your application is deployed you can run:
{{< highlight bash >}}
convox apps info -a  <TEMP_APP_NAME>
{{< / highlight >}}
And see some new urls you can test your application with. I would recommend ensuring these urls are working correctly. 

If you are serving your application on a custom domain you can now change it's dns settings. One of the nice things about gen 2 is instead of pointing a domain at a specific application endpoint
you actually point it at the rack's ALB domain. You can find this value by doing:
{{< highlight bash >}}
➜  ~ convox rack
Name    <MY_RACK>
Status   running
Version  20171213132441
Count    3
Domain   <MY_RACK_DOMAIN>
Region   us-west-2
Type     t2.small
{{< / highlight >}}

Once the you've made the dns changes, wait the required amount of time and using whatever access logs you have available, validate traffic is going to your new gen 2 application.

**Note:** Internal applications use their own ALB, and if you want to use custom domains on internal applications you must find the ALB's domain name in AWS. It is currently not listed when running `convox rack`

### Cleanup
You most likely want to keep the name of the previous application and not have to versions running. Luckily we can easily migrate by deleting the old application and creating a new one just like we did above with the old name.

{{< highlight bash >}}
 convox apps delete <APP_NAME>
{{< / highlight >}}
Wait isn't supported on convox apps delete, so you'll have to wait and monitor `convox apps` until it disappears

{{< highlight bash >}}
convox apps create <APP_NAME> --generation 2 --wait
convox env -a <TEMP_APP_NAME> | convox env set -a <APP_NAME> --wait
convox deploy -a <APP_NAME> --wait
convox apps delete <TEMP_APP_NAME>
{{< / highlight >}}

Because we pointed the domain at the rack domain there is no need to change DNS settings again.

And now you should have a new shiny gen 2 application!

# A couple of gotchas
## ALBs take advantage of the Host header to handle routing.
Personally, this was my first experience with using ALBs, so I didn't really know much about how they worked. Turns out they actually take 
advantage of the Host header to determine what incoming domain should route to which specific target group. While this seems obvious, that's not how ELBs work. and in
For for my application I run gunicorn server serving Django with a nginx* reverse proxy in front of it, and I was using something like:

{{< highlight conf >}}
    location / {
      # checks for static file, if not found proxy to app
      try_files $uri @proxy_to_app;
    }
    
    location @proxy_to_app {
      set_by_lua $django_url 'return os.getenv("DJANGO_URL")';
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header Host $http_host;
      proxy_redirect off;
      proxy_pass $django_url;
    }
{{< / highlight >}}

In my nginx.conf. Having that proxy `proxy_set_header Host $http_host;` line actually breaks ALB's ability to route the request. 
In addition to that, I had some problems with using `proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;` as well, in some situations resulting in a 462 http error. 
If you have anything like that setup I would recommend just removing those lines.

*Probably overkill, but I actually use openresty so that I can inject env variables into the config. 
