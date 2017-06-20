# Open Source Cloud Native workloads on LinuxONE

Open source software has expanded from a low-cost alternative to a platform for enterprise databases, clouds and next-generation apps. These workloads need higher levels of scalability, security and availability from the underlying hardware infrastructure.

LinuxONE was built for open source so you can harness the agility of the open revolution on the industry’s most secure, scalable and high-performing Linux server. In this journey we will show how to run open source Cloud-Native workloads on LinuxONE

## Scenarios

1. [Use Docker images from Docker hub to run your workloads on LinuxONE](#scenario-one-use-docker-images-from-docker-hub-to-run-your-workloads-on-linuxone)
1.1 [WordPress](#1-install-and-run-wordpress)
1.2 [WebSphere Liberty](#2-install-and-run-websphere-liberty)
2. Create your own Docker images for LinuxONE](#scenario-two-create-your-own-docker-images-for-linuxone)
2.1 [GitLab]()
3. [Use Kubernetes on LinuxONE to run your cloud-naive workloads](#scenario-three-use-kubernetes-on-linuxone-to-run-your-cloud-naive-workloads)

## Included Components

- [LinuxONE](https://www-03.ibm.com/systems/linuxone/open-source/index.html)
- [Docker](https://www.docker.com)
- [Docker Store](https://sore.docker.com)
- [WordPress](https://workpress.com)
- [MariaDB](https://mariadb.org)

## Prerequisites

Register at [LinuxONE Communinity Cloud](https://developer.ibm.com/linuxone/) for a trial account.
We will be using a Ret Hat base image for this journey, so be sure to chose the
'Request your trial' button on the left side of this page.

## Scenario One: Use Docker images from Docker hub to run your workloads on LinuxONE

[Docker Hub](https://hub.docker.com) makes it rather simple to get started with containers, as there are quite a few images ready to for your to use.  You can browse the list of images that are compatable with LinuxONE by doing a search on the ['s390x'](https://hub.docker.com/search/?isAutomated=0&isOfficial=0&page=1&pullCount=0&q=s390x&starCount=0) tag. We will start off with everyone's favorite demo: an installation of WordPress.

These instructions assume a base RHEL 7.2 image. 

### Install docker
```text
:~$ yum install docker.io
```

### Install docker-compose

Install dependencies

```text
sudo yum install -y python-setuptools
```

Install pip with easy_install

```text
sudo easy_install pip
```

Upgrade backports.ssl_match_hostname

```text
sudo pip install backports.ssl_match_hostname --upgrade
```

Finally, install docker-compose itself
```text
sudo pip install docker-compose
```

### 1. Install and run WordPress

### 2. Install and run WebSphere Liberty

In this step, we will once again be using existing images from Docker Hub - this time to set up a WebSphere Application Server.  We will be implementing it for Java EE7 Full Platform compliance.

### 1. Setup

Our implemetation of WebSphere will be based off the [application deployment sample (https://developer.ibm.com/wasdev/docs/article_appdeployment/), which means we will first need to download the DefaultServletEngine sample and extract it to `/tmp`:

```text
wget https://github.com/WASdev/sample.servlet/releases/download/V1/DefaultServletEngine.zip
unzip DefaultServletEngine.zip -d /tmp/DefaultServletEngine
```
We will also need to modify the server.xml file to accept HTTP connections from outside of the container:

```text
vim server.xml
```
Find the `server` stanza and add the following:
```text
<httpEndpoint host="*" httpPort="9080" httpsPort="-1"/>
```

### 2. Docker Run

Now run the container

```text
$ docker run -d -p 80:9080 -p 443:9443 \
  -v /tmp/DefaultServletEngine/dropins/Sample1.war:/config/dropins/Sample1.war \
  websphere-liberty:webProfile7
```

### 3. Browse

Once the server is started, you can browse to
`http://localhost/Sample1/SimpleServlet` on the Docker host.


## Scenario Two: Create your own Docker images for LinuxONE

In our previous scenario, we used a couple of container images that had already been created and were waiting for our use in the Docker Hub Community.  But what if you are looking to run a workload that is not currently available there?  In this scenario, we will walk through the steps to create your own Docker images.  It helps to start with a base OS image; in this case we will be using Ubuntu ([s390x/ubuntu](https://hub.docker.com/r/s390x/ubuntu/)). On top of which, we will use very popular repository, GitLab for this example.

### 1. Setup

First, let's create a directory for our scenario:

```text
$ mkdir gitlabexercise
$ cd gitlabexercise
```

You will also need a `requirements.txt` file in the directory with the contents:

```text
gitlab
postgres
redis
```

### 2. Create Dockerfiles

Next we need to write a few Dockerfiles to build our Docker images.  In the
project directory, create the following three files with their respective
content:

Dockerfile-gitlab
```text
FROM s390x/ubuntu

# update & upgrade the ubuntu base image
RUN apt-get update && apt-get upgrade -y

# Install required packages
RUN apt-get update -q \
    && DEBIAN_FRONTEND=noninteractive apt-get install -yq --no-install-recommends \
      ca-certificates \
      openssh-server \
      wget \
      apt-transport-https \
      vim \
      apt-utils \
      curl \
      postfix \
      nano

# Install the gitlab 
RUN apt-get install -y gitlab

# Manage SSHD through runit
RUN mkdir -p /opt/gitlab/sv/sshd/supervise \
    && mkfifo /opt/gitlab/sv/sshd/supervise/ok \
    && printf "#!/bin/sh\nexec 2>&1\numask 077\nexec /usr/sbin/sshd -D" > /opt/gitlab/sv/sshd/run \
    && chmod a+x /opt/gitlab/sv/sshd/run \
    && ln -s /opt/gitlab/sv/sshd /opt/gitlab/service \
    && mkdir -p /var/run/sshd

# Disabling use DNS in ssh since it tends to slow connecting
RUN echo "UseDNS no" >> /etc/ssh/sshd_config

# Prepare default configuration
RUN ( \
  echo "" && \
  echo "# Docker options" && \
  echo "# Prevent Postgres from trying to allocate 25% of total memory" && \
  echo "postgresql['shared_buffers'] = '1MB'" ) >> /etc/gitlab/gitlab.rb && \
  mkdir -p /assets/ && \
  cp /etc/gitlab/gitlab.rb /assets/gitlab.rb

# Expose web & ssh
EXPOSE 443 80 22

# Define data volumes
VOLUME ["/etc/gitlab", "/var/opt/gitlab", "/var/log/gitlab"]

# Copy assets
COPY assets/wrapper /usr/local/bin/

# Wrapper to handle signal, trigger runit and reconfigure GitLab
CMD ["/usr/local/bin/wrapper"]
```

Dockerfile-postgres

```text
#
# example Dockerfile for https://docs.docker.com/examples/postgresql_service/
#

FROM s390x/ubuntu

RUN apt-get update && apt-get upgrade -y

# Add the PostgreSQL PGP key to verify their Debian packages.
# It should be the same key as https://www.postgresql.org/media/keys/ACCC4CF8.asc
RUN apt-key adv --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys B97B0AFCAA1A47F044F244A07FCC7D46ACCC4CF8

# Install ``python-software-properties``, ``software-properties-common`` and PostgreSQL
#  There are some warnings (in red) that show up during the build. You can hide
#  them by prefixing each apt-get statement with DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y python-software-properties software-properties-common postgresql postgresql-client postgresql-contrib

# Note: The official Debian and Ubuntu images automatically ``apt-get clean``
# after each ``apt-get``

# Run the rest of the commands as the ``postgres`` user created by the ``postgres`` package when it was ``apt-get installed``
USER postgres

# Create a PostgreSQL role named ``docker`` with ``docker`` as the password and
# then create a database `docker` owned by the ``docker`` role.
# Note: here we use ``&&\`` to run commands one after the other - the ``\``
#       allows the RUN command to span multiple lines.
RUN    /etc/init.d/postgresql start &&\
    psql --command "CREATE USER docker WITH SUPERUSER PASSWORD 'docker';" &&\
    createdb -O docker docker

# Adjust PostgreSQL configuration so that remote connections to the
# database are possible.
RUN echo "host all  all    0.0.0.0/0  md5" >> /etc/postgresql/9.5/main/pg_hba.conf

# And add ``listen_addresses`` to ``/etc/postgresql/9.5/main/postgresql.conf``
RUN echo "listen_addresses='*'" >> /etc/postgresql/9.5/main/postgresql.conf

# Expose the PostgreSQL port
EXPOSE 5432

# Add VOLUMEs to allow backup of config, logs and databases
VOLUME  ["/etc/postgresql", "/var/log/postgresql", "/var/lib/postgresql"]

# Set the default command to run when starting the container
CMD ["/usr/lib/postgresql/9.5/bin/postgres", "-D", "/var/lib/postgresql/9.5/main", "-c", "config_file=/etc/postgresql/9.5/main/postgresql.conf"]
```

Dockerfile-redis

```text
FROM        s390x/ubuntu
RUN         apt-get update && apt-get upgrade -y && apt-get install -y redis-server
EXPOSE      6379
ENTRYPOINT  ["/usr/bin/redis-server"]
```

### 3. Define service in a Compose file

Again, we are going to use docker-compose to manage our Docker images.  In the
project directory, create a `docker-compose.yml` file that contains:

### 4. Build and run

```text
$ docker-compose up
```

## Scenario Three: Use Kubernetes on LinuxONE to run your cloud-naive workloads

To begin with, we will set up a Kubernetes cluster.  A cluster requires a number
of containers, namely: an `apiserver`, a `scheduler`, a `controller` and a
`proxy` service.  We will start these individually.

### 1. Start the etcd service
```text
docker run -d \
--net=host \
--name=etcd \
brunswickheads/kubernetes-s390x etcd --addr=127.0.0.1:4001 --bind-addr=0.0.0.0:4001 --data-dir=/var/etcd/data
```

### 2. Start kubelet service
```text
docker run \
--volume=/:/rootfs:ro \
--volume=/sys:/sys:ro \
--volume=/var/lib/docker/:/var/lib/docker:ro \
--volume=/var/data/kubelet:/var/data/kubelet/:rw \
--volume=/var/run:/var/run:rw \
--net=host \
--privileged=true \
-d \
--name=kubelet \
brunswickheads/kubernetes-s390x:latest \
hyperkube kubelet --containerized --root-dir=/var/data/kubelet --hostname-override="127.0.0.1" --address="0.0.0.0" --api-servers=http://localhost:8080 --config=/etc/kubernetes/manifests
```

### 3. Start apiserver service
```text
docker run \
--volume=/:/rootfs:ro \
--volume=/sys:/sys:ro \
--volume=/var/lib/docker/:/var/lib/docker:ro \
--volume=/var/data/kubelet:/var/data/kubelet/:rw \
--volume=/var/run:/var/run:rw \
--net=host \
--privileged=true \
-d \
--name=apiserver \
brunswickheads/kubernetes-s390x:latest \
hyperkube apiserver --portal_net=10.0.0.1/24 --address=0.0.0.0 --insecure-bind-address=0.0.0.0 --etcd_servers=http://127.0.0.1:4001 --cluster_name=kubernetes --v=2
```

### 4. Start controller service
```text
docker run \
-d \
--net=host \
-v /var/run/docker.sock:/var/run/docker.sock \
--name=controller \
brunswickheads/kubernetes-s390x:latest hyperkube controller-manager --master=127.0.0.1:8080 --v=2
```

### 5. Start scheduler service
```text
docker run \
-d \
--net=host \
-v /var/run/docker.sock:/var/run/docker.sock \
--name=scheduler \
brunswickheads/kubernetes-s390x:latest hyperkube scheduler --master=127.0.0.1:8080 --v=2
```

### 6. Start proxy service
```text
docker run \
-d \
--net=host \
--privileged \
--name=proxy \
brunswickheads/kubernetes-s390x:latest hyperkube proxy --master=http://127.0.0.1:8080 --v=2
```

### 7. Create a pause imaging by adding a tag to the kubernetes image:
```text
docker tag <k8s_image_name>:latest gcr.io/google_containers/pause:0.8.0
```

