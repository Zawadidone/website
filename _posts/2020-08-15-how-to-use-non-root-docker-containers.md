---
title: Why You Should Be Using Non Root Docker Containers
name: Why You Should Be Using Non Root Docker Containers
date: '2020-08-15'
header:
  teaser: "/assets/img/Screenshot 2020-08-15 at 14.03.42.png"
---

I am assuming you are already familiar with Docker.  What most of the people do when using official Docker images is pull the image, install some stuff and let the container run a command like `nginx`.  This means that the started process has the highest privileges on the server because root `uid 0` in a container is the same as on the host. This is not necessary for using a software package in a container.


In this example, I show the wrong way of using the NGINX image.

```bash
docker run -it --rm nginx:alpine /bin/sh
# id
uid=0(root) gid=0(root) groups=0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel),11(floppy),20(dialout),26(tape),27(video)
# nginx
# ps aux
PID   USER     TIME  COMMAND
    1 root      0:00 /bin/sh
    9 root      0:00 nginx: master process nginx
```

Instead of running the process `nginx` as a non-root user the process is started by the user `root`.  This happens because the official Docker images always provides the `root` user so the developer can install packages and do other stuff just like on a server when installing a software package or doing updates. A lot of the official docker images also provide a non-root user which almost always has the name of the software package. To use this non-root user Docker has the `USER` instruction that switches the user in the Docker image. For this example I used the Nginx image.

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

Run the image

```bash
docker build -t nginx .
docker run -it --rm nginx /bin/sh
$ id
uid=101(nginx) gid=101(nginx) groups=101(nginx)
$ nginx
2020/08/14 14:35:27 [warn] 10#10: the "user" directive makes sense only if the master process runs with super-user privileges, ignored in /etc/nginx/nginx.conf:2
nginx: [warn] the "user" directive makes sense only if the master process runs with super-user privileges, ignored in /etc/nginx/nginx.conf:2
$ ps aux
PID   USER     TIME  COMMAND
    1 nginx     0:00 /bin/sh
   11 nginx     0:00 nginx: master process nginx
```

In this example, I set some file permissions for the user `nginx` and then use the Docker instruction `USER` to switch. Now when running the Docker image the process `nginx` is started as the non-root user `nginx` instead of `root`.


In my Github repository called [non-root-images](https://github.com/Zawadidone/non-root-docker-images) I have made Dockerfile examples for the software packages NGINX, Redis,  Python, PHP and Node JS.
