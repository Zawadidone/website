---
title: Using Docker images from scratch
name: Using Docker images from scratch
date: '2022-01-12 08:33:23'
header:
  teaser: "/assets/img/docker-image-scratch.png"
---

The last blog post about Docker was about using [non-root Docker containers](https://zawadidone.nl/how-to-use-non-root-docker-containers/) and why this is safer. This time I want to go a step further and explain what I think is one of the best Docker features called image `FROM` [scratch](https://hub.docker.com/_/scratch). This feature allows creating a new [empty layer](https://docs.docker.com/develop/develop-images/baseimages/) in your image.


Instead of adding your application to for example a Ubuntu image that has the size of more than 50MB and contains almost a full Linux distribution without the Linux kernel. Most developers use [Alpine](https://www.alpinelinux.org/) images that are minimal but still contain unnecessary files and folders.

```bash
docker images
REPOSITORY TAG      IMAGE ID        CREATED          SIZE
ubuntu     20.04    f63181f19b2f    11 months ago    72.9MB
alpine     3.15     c059bfaa849c    6 weeks ago      5.59MB

docker run -it --rm  ubuntu:20.04 ls
bin  boot  dev  etc  home  lib  lib32  lib64  libx32  media  mnt  opt  proc  root  run  sbin  srv  sys  tmp  usr  var

docker run -it --rm alpine:3.15 ls
bin    etc    lib    mnt    proc   run    srv    tmp    var dev    home   media  opt    root   sbin   sys    usr
```

In this example, I will compare a minimal NGINX container with an NGINX container from scratch. But this can be done with any application.  My preferred way of creating images is using the distribution [Alpine](https://www.alpinelinux.org/) which is minimal and if possible using a scratch layer that only contains the application, libraries and configuration files. If the application can be compiled statically libraries are not even needed.

### Minimal
The official NGINX alpine image.

```Dockerfile
FROM nginx:alpine

# copy Nginx config files
COPY default.conf /etc/nginx/conf.d/
COPY nginx.conf /etc/nginx/

# set file permissions for nginx user
RUN chown -R nginx:nginx /var/cache/nginx /etc/nginx/

# switch to non-root user
USER nginx

CMD ["nginx", "-g", "daemon off;"]
```
### From scratch
Before creating an image from scratch all the necessary files, libraries and configuration files need to be known. `ldd` is a tool that prints the shared libraries required by each program. Those libraries shown below are needed in the image from scratch.

```bash
docker run -it --rm -p 8080:8080 nginx:minimal /bin/sh
/ $ which nginx
/usr/sbin/nginx
/ $ ldd /usr/sbin/nginx
	/lib/ld-musl-x86_64.so.1 (0x7f2bf1cfc000)
	libpcre2-8.so.0 => /usr/lib/libpcre2-8.so.0 (0x7f2bf1b1a000)
	libssl.so.1.1 => /lib/libssl.so.1.1 (0x7f2bf1a99000)
	libcrypto.so.1.1 => /lib/libcrypto.so.1.1 (0x7f2bf1818000)
	libz.so.1 => /lib/libz.so.1 (0x7f2bf17fe000)
	libc.musl-x86_64.so.1 => /lib/ld-musl-x86_64.so.1 (0x7f2bf1cfc000)
```

The same NGINX is used as a base for the scratch image. By copying (`COPY --from=base`) files from the base layer to the scratch layer the image from scratch is created. The following can also be done with compiled and scripted programming languages. But to make it simple I used NGINX as an example.


```Dockerfile
FROM nginx:alpine as base

RUN chown -R nginx:nginx /var/cache/nginx /etc/nginx/

FROM scratch

COPY --from=base /etc/passwd /etc/passwd

copy --from=base [ \
    "/lib/ld-musl-x86_64.so.1", \
    "/lib/libssl.so.1.1", \
    "/lib/libcrypto.so.1.1", \
    "/lib/libz.so.1", \
    "/lib/ld-musl-x86_64.so.1", \
    "/lib/" \
    ]

copy --from=base ["/usr/lib/libpcre2-8.so.0", "/usr/lib/"]

copy --from=base ["/usr/sbin/nginx", "/usr/sbin/nginx"]
copy --from=base ["/var/log/nginx", "/var/log/nginx"]
copy --from=base ["/etc/nginx", "/etc/nginx"]
copy --from=base ["/usr/share/nginx/html/index.html", "/usr/share/nginx/html/index.html"]

COPY default.conf /etc/nginx/conf.d/
COPY nginx.conf /etc/nginx/

USER nginx

CMD ["/usr/sbin/nginx", "-g", "daemon off;"]
```

Comparing the size of the image shows that the image from scratch is only 5MB compared to 23MB with a minimal Alpine image. This makes it possible to deploy faster. This makes a huge difference with a Kubernetes cluster with hundreds of containers

```bash
docker images
REPOSITORY TAG      IMAGE ID       CREATED          SIZE
nginx      scratch  67df498a2d83   42 seconds ago   5.68MB
nginx      minimal  51df82266e84   55 minutes ago   23.5MB
```

### Security advice
The National Security Agency (NSA) and the Cybersecurity and Infrastructure Security Agency (CISA) released a Cybersecurity Technical Report [Kubernetes Hardening Guidance](https://media.defense.gov/2021/Aug/03/2002820425/-1/-1/1/CTR_KUBERNETES%20HARDENING%20GUIDANCE.PDF). Which also contains advisories about images from scratch.

**Building secure container images**
> Container images are usually created by either building a container from scratch or by
building on top of an existing image pulled from a repository. In addition to using trusted
repositories to build containers, image scanning is key to ensuring deployed containers
are secure. Throughout the container build workflow, images should be scanned to
identify outdated libraries, known vulnerabilities, or misconfigurations, such as insecure
ports or permissions. - [Kubernetes Hardening Guidance](https://media.defense.gov/2021/Aug/03/2002820425/-1/-1/1/CTR_KUBERNETES%20HARDENING%20GUIDANCE.PDF)

By using an image from scratch a container doesn't include  [GTFOBins](https://gtfobins.github.io/) attackers can use when the container is compromised.

**Immutable container file systems**
> By default, containers are permitted mostly unrestricted execution within their own
context. A cyber actor who has gained execution in a container can create files,
download scripts, and modify the application within the container. Kubernetes can lock
down a container’s file system, thereby preventing many post-exploitation activities.
However, these limitations also affect legitimate container applications and can
potentially result in crashes or anomalous behavior. To prevent damaging legitimate
applications, Kubernetes administrators can mount secondary read/write file systems for
specific directories where applications require write access. Appendix B: Example
deployment template for read-only filesystem shows an example immutable
container with a writable directory. - [Kubernetes Hardening Guidance](https://media.defense.gov/2021/Aug/03/2002820425/-1/-1/1/CTR_KUBERNETES%20HARDENING%20GUIDANCE.PDF)

The use of a read-only file system makes it harder for an attacker to use the file system as a playground.

Besides the security advantages, I don't understand this feature is not used more in the Docker community, because it shows what Docker is good at. Using a tool that make it easier to package an application in a container without the need for other tools. By using an image from scratch the Docker container can be used like an executable with configuration files. Instead of a container with a full Linux distribution (overhead) without Linux kernel.
### Appendix
The used NGINX configuration.

The temporary paths use `/dev/shm` instead of `/var/cache/nginx` to make it possible to use a read-only file system.

/etc/nginx/nginx.conf
```
cat nginx.conf
user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log warn;
pid        /dev/shm/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;
    sendfile        on;
    keepalive_timeout  65;

    client_body_temp_path /dev/shm/client_temp;
    proxy_temp_path /dev/shm/proxy_temp;
    fastcgi_temp_path /dev/shm/fastcgi_temp;
    uwsgi_temp_path /dev/shm/uwsgi_temp;
    scgi_temp_path /dev/shm/scgi_temp;

    include /etc/nginx/conf.d/*.conf;
}
```

/etc/nginx/conf.d/default.conf
```
server {
    listen       8080;
    server_name  localhost;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```
