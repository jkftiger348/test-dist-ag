# Kubernetes and MMR Replication Clusters



## Introduction

We discuss running a MMR (repl) replication cluster inside Kubernetes

MMR replication clusters are different from distributed  AG clusters in these important ways:

1. Each member of the cluster needs to be able to do a TCP connection to each other member of the cluster.   The connection is to a port computed at run time.  The range of port numbers to which a connection is made can be constrained by the agraph.cfg file but typically this will be a large range to ensure that at least one port in that range is not in used.
2. All members of the cluster hold the complete database (although for brief periods of time they can be out of sync and catching up with one another).

MMR replication clusters don't quite fit the k8s model in these ways

1. When the cluster is running normally each instance knows the DNS name or IP address of each other instance.    In k8s you don't want to depend on the IP address of another cluster's pod as those pods can go away and  a replacement started at a different IP address.   We'll describe below our solution to this.
2. Services are a way to hide the actual location of a pod however they are designed to handle a set of known ports..  In our case we need to connect from one pod to a known-at-runtime port of another pod and this isn't what services are designed for.
3. A key feature of k8s is the ability to scale up and down the number of processes in order to handle the load appropriately.



## The Design

1. We have one instance called the controlling instance.   We use a StatefulSet to control this instance which guarantees that its name is well known and in fact is always 'controlling-0' since there is exactly one instance and the naming is zero based.
2. We have a headless service for our controlling instance StatefulSet and that causes there to be a DNS entry for 'controlling' that points to the current IP address of the node in which the controlling instance runs.   Thus we don't need to hardwire-in the IP address of the controlling instance (as we do in our AWS load balancer implementation).
3. The controlling instance uses two PersistentVolumes to store: 1. The repo we're replicating and 2. The token that other nodes can use to connect to this node. Should the controlling instance agraph server die (or the pod in which it runs dies) then when the pod is started again it will have access to the data on those two persistent volumes.
4. We call the other instances  in the cluster Copy instances.   These are full read-write instances of the repo but we don't back up their data in a persistent volume.  This is because we want to scale up and down the number of Copy instances.  When we scale down we don't want to save the old data since when we scale down we remove that instance from the cluster thus the repo in the cluster can never join the cluster again.   We denote the copy instances by their IP addresses.  The Copy  instances can find the address of the controlling instance via DNS.  The controlling instance will pass the cluster configuration to the copy instance and that configuration information will given the IP addresses of the other Copy instances.  This is how the Copy instances find each other.
5. We have a load balancer that allows one to access a random Copy instance from an external IP address.   This load balancer doesn't support sessions so it's only useful for doing queries and quick inserts that don't need a session.
6. We have a load balancer that allows one to access the Controlling instance.  While this load balancer also doesn't have session support, because there is only one controlling instance it's not a problem if you start an agraph session because all sessions will live on the single controlling instance.



## Implementation

We build and deploy in three subdirectories

### Directory ag/


In this directory we build a container holding an installed agraph.    The Dockerfile is


```
FROM centos:7

# using centos 6 requires that a change be made
# in ../agrepl/vars.sh since
# ifconfig prints addresses differently
#
# agraph root is /app/agraph
#

RUN yum -y install net-tools iputils bind-utils wget

ARG agversion=agraph-6.6.0
ARG agdistfile=${agversion}-linuxamd64.64.tar.gz

ADD ${agdistfile} .

# needed for agraph 6.7.0 and can't hurt for others
# change to =11 if you only hae OpenSSL 1.1 installed.
ENV ACL_OPENSSL_VERSION=10

# optional step.
# eliminate fancy prompts that don't work inside an emacs shell window
ENV PROMPT_COMMAND=

RUN groupadd agraph && useradd -d /data -g agraph agraph

RUN mkdir /app

RUN (cd ${agversion} ;  ./install-agraph /app/agraph -- --non-interactive \
                --runas-user agraph \
                --super-user test \
                --super-password xyzzy )

# stuff we don't need
RUN rm -fr /app/agraph/lib/doc /app/agraph/lib/demos

# we will attach persistent storage to this
RUN mkdir /app/agraph/data/rootcatalog

# patch to reduce cache time so we'll see when the controlling instance moves

COPY dnspatch.cl /app/agraph/lib/patches/dnspatch.cl

RUN chown -R agraph.agraph /app/agraph
```


The Dockerfile installs agraph in /app/agraph and creates a super user with the name test and the password xyzzy.    It creates a user 'agraph' so that the agraph server will run as the user agraph rather than as root.

We have to worry about the controlling instance process dying and being restarted in another pod with a different IP address.   Thus if we've cached the DNS mapping of 'controlling' we need to notice as soon as possible that the mapping as changed.  The dnspatch.cl file changes a parameter in the acl DNS code to reduce the time we trust our DNS cache to be accurate.

We also install a number of networking tools.   Agraph doesn't need these but if we want to do debugging inside the container they are useful to have installed.

This container is pushed to the docker hub as jkftiger/ag:latest.    jkftiger is a dockerhub account name.   You'll want to use your own dockerhub account name if wish to run this code..



## Directory agrepl/

Next we take the container created above and add the specific code to support replication clusters.

The Dockerfile is

```
FROM jkftiger/ag:latest

#
# Agraph root is /app/agraph

RUN mkdir /app/agraph/scripts
COPY . /app/agraph/scripts

# since we only map one port from the outside into our cluster
# we need any sessions created to continue to use that one port.
RUN echo "UseMainPortForSessions true" >> /app/agraph/lib/agraph.cfg

# setting/user will be overwritten with a persistent mount so copy
# the data to another location so it can be restored.

RUN cp -rp /app/agraph/data/settings/user /app/agraph/data/user

ENTRYPOINT ["/app/agraph/scripts/repl.sh"]
```


This Dockerfile uses  jkftiger/ag:latest and when we process
the Dockerfile we give it the tag jkftiger/agrepl:latest

The Dockerfile installs the scripts repl.sh and vars.sh.
These are run when this container starts.

We modify the agraph.cfg with a line that ensures that even if we create a session that we'll continue to access it via port 10035 since the load balancer we'll use to access agraph only forwards 10035 to agraph.

Also we know that we'll be installing a persistent volume at /app/agraph/data/user so we make a copy of that directory in another location since the current contents will be invisible when a filesystem is mounted on top of it.   We need the contents as that is where the credentials for the user *test* are stored.

 Initially the file settings/user/test will contain the credentials we specified when we installed agraph in first Dockerfile.   When we create a cluster instance a new token is created and this is used in place of the password for the test account.  This token is stored in settings/user/test which is why we need this to be an instance-specific and persistent filesystem for the controlling instance.

When this container starts it runs repl.sh which first runs vars.sh

vars.sh is

```
# constants need by scripts

port=10035
authuser=test
authpassword=xyzzy
reponame=myrepl

# compute our ip address
# centos 6
#myip=$(ifconfig eth0 | grep inet | awk "{print substr(\$2,6)}")
#
# centos 7 and fedora

myip=$(ifconfig eth0 | grep "inet " | awk "{print (\$2)}")
```


In vars.sh we specify the information about the repository we'll create and the authentication we'll use to access the agraph server.    This data must match what we used when we installed the agraph server in the first Dockerfile.

The script repl.sh is this:


```
#!/bin/bash
#
## to start ag and then create or join a cluster
##

cd /app/agraph/scripts

set -x
. ./vars.sh
echo ip is $myip

# move the copy of user with our login to the newly mounted volume
# if this is the first time we've run agraph on this volume
if [! -e /app/agraph/data/rootcatalog/$reponame] ; then
    cp -rp /app/agraph/data/user/\* /app/agraph/data/settings/user
fi

# due to volume mounts /app/agraph/data could be owned by root
# so we have to take back ownership
chown -R agraph.agraph /app/agraph/data

## start agraph
/app/agraph/bin/agraph-control --config /app/agraph/lib/agraph.cfg start

sleep 2

term_handler() {
   # this signal is delivered when the pod is
   # about to be killed.  We remove ourselves
   # from the cluster.
   echo got term signal
   /bin/bash -c ./remove-instance.sh
   exit
}

sleepforever() {
    # This unusual way of sleeping allows
    # a TERM signal sent when the pod is to
    # die to then cause the shell to invoke
    # the term_handler function above.
    date
    while true
    do
        sleep 99999 & wait ${!}
    done
}

if [-e /app/agraph/data/rootcatalog/$reponame] ; then
    echo database already exists in this persistent volume
    sleepforever
fi

controllinghost=controlling

if [x$Controlling == "xyes"] ;
then
   ## create first and controlling cluster instance
   curl -s -X PUT -u $authuser:$authpassword "http://127.0.0.1:$port/repositories/$reponame/repl/createCluster?host=$controllinghost&port=$port&user=$authuser&password=$authpassword&instanceName=controlling"
else
    sleep 5
    
    # wait for the controlling ag server to be running
    until curl -s http://$authuser:$authpassword@$controllinghost:$port/hostname ; do echo wait for agserver controlling ; sleep 15; done

   # give the repo time to be created and converted into a replication
   # instance.

   sleep 15

   # wait for cluster repo on the controlling instance to be present
   until curl -s http://$authuser:$authpassword@$controllinghost:$port/repositories/$reponame/repl/status ; do echo wait for repo ; sleep 15; done

   myiname=i-$myip
   echo $myiname > instance-name.txt

   # construct the remove-instance.sh shell script to remove this instance
   # from the cluster when the instance is terminated.

   echo curl -s -X PUT -u $authuser:$authpassword "http://$controllinghost:$port/repositories/$reponame/repl/remove?instanceName=$myiname" > remove-instance.sh

chmod 755 remove-instance.sh

   #
   trap term_handler SIGTERM SIGHUP SIGUSR1
   trap -p
   echo this pid is $$

   # join the cluster
   echo joining the cluster
   curl -s -X PUT -u $authuser:$authpassword "http://$controllinghost:$port/repositories/$reponame/repl/growCluster?host=$myip&port=$port&name=$reponame&user=$authuser&password=$authpassword&instanceName=$myiname"

fi

sleepforever
```

This script can be run under three different conditions

1. Run when the Controlling instance is starting for the first time
2. Run when the Controlling instance is restarting having run before and died (perhaps the machine on which it was running crashed or the agraph process had some error)
3. Run when a Copy instance is starting for the first time.   Copy instances are not restarted when they die.  Therefore we don't need to handle the case of a Copy instance restarting.



In cases 1 and 2 the environment variable Controlling will have the value "yes".

In case 2 there will be a directory at /app/agraph/data/rootcatalog/$reponame

In all cases we start an agraph server.

In case 1 we create a new cluster.   In case 2 we just sleep and let the agraph server recover the replication repo and reconnect to the other members of the cluster.

In case 3 we wait for the controlling instance's agraph to be running.  Then we wait for the replication repo we want to copy to be up and running.  Then we grow the cluster by copying the cluster repo.

We also create a script which will remove this instance from the cluster  should this pod be terminated.   When the pod is killed (likely due to us scaling down the number of Copy instances) a termination signal will be sent first to the process allowing it to run this remove script before the pod completely disappears.



## Directory kube/

This directory contains the yaml files that create kubernetes resources when then create pods and start the containers that create the agraph replication cluster.

### controlling-service.yaml

We begin by defining the services.   It may seem to make sense to define the applications before defining the service to expose the application but it's the service we create that puts the application's address in DNS and we want don't want there to be a time when the application is running yet it's not in DNS yet.
```
apiVersion: v1
kind: Service
metadata:
 name: controlling
spec:
 clusterIP:  None
 selector:
   app: controlling
 ports:
 - name: http
   port: 10035
   targetPort: 10035
```




This selector defines a service for any container with a label with a key "app" and a value "controlling".  There aren't any yet but there will be.  You create this service with

```
% kubectl create -f controlling-service.yaml
```

In fact for all the yaml files shown below you create the object they define by running

```
% kubectl create -f  filename.yaml
```

### copy-service.yaml

We do a similar service for all the copy applications.


```
apiVersion: v1
kind: Service
metadata:
 name: copy
spec:
 clusterIP: None
 selector:
   app: copy
 ports:
 - name: main
   port: 10035
   targetPort: 10035
```




### controlling.yaml

This is the most complex resource description for the cluster.
We use a StatefulSet so we have a predictable name for the single pod we create.   We define two persistent volumes.  A StatefulSet is designed to control more than one pod so rather than a VolumeClaim we have a VolumeClaimTemplate so that each Pod can have its own persistent volume… but as it turns out we only one one pod in this set and we never scale up.  There must be exactly one controlling instance.   We setup a liveness check so that if the agraph server dies we'll restart the pod and thus the agraph server.

We set the environment variable "Controlling" to "yes" and this causes this container to start up as a controlling instance.


```
#
# stateful set of controlling instance
#

apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: controlling
spec:
  serviceName: controlling
  replicas: 1
  template:
    metadata:
      labels:
        app: controlling
    spec:
        containers:
        - name: controlling
          image: jkftiger/agrepl:latest
          imagePullPolicy: Always
          livenessProbe:
            httpGet:
              path: /hostname
              port: 10035
            initialDelaySeconds: 30
          volumeMounts:
          - name: shm
            mountPath: /dev/shm
          - name: data
            mountPath: /app/agraph/data/rootcatalog
          - name: user
            mountPath: /app/agraph/data/settings/user
          env:
          - name: Controlling
            value: "yes"
        volumes:
         - name: shm
           emptyDir:
             medium: Memory
  volumeClaimTemplates:
         - metadata:
            name: data
           spec:
            resources:
              requests:
                storage: 20Gi
            accessModes:
            - ReadWriteOnce
         - metadata:
            name: user
           spec:
            resources:
              requests:
                storage: 10Mi
            accessModes:
            - ReadWriteOnce
```

### copy.yaml

This StatefulSet is responsible for starting all the other instances.    It's much simpler as it doesn't use Persistent Volumes

```
#
# stateful set of copies of the controlling instance
#

apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: copy
spec:
  serviceName: copy
  replicas: 2
  template:
    metadata:
      labels:
        app: copy
    spec:
        volumes:
         - name: shm
           emptyDir:
             medium: Memory
        containers:
        - name: controlling
          image: jkftiger/agrepl:latest
          imagePullPolicy: Always
          livenessProbe:
            httpGet:
              path: /hostname
              port: 10035
            initialDelaySeconds: 30
          volumeMounts:
          - name: shm
            mountPath: /dev/shm
```


### controlling-lb.yaml

We define a load balancer so applications on the internet outside of our cluster can communicate with the controlling instance.    The IP address of the load balancer isn't specified here.   The cloud service provider (i.e. Google Cloud Platform or AWS) will determine an address after a minute or so and will make that value visible if you run

```
% kubectl get svc controlling-loadbalancer
```

The file is

```
apiVersion: v1
kind: Service
metadata:
  name: controlling-loadbalancer
spec:
  type: LoadBalancer
  ports:
  - port: 10035
    targetPort: 10035
  selector:
    app: controlling
```


### copy-lb.yaml

As noted earlier the load balancer for the copy instances does not support sessions   However you  can use the load balancer to issue queries or simple inserts that don't require a session.


```
apiVersion: v1
kind: Service
metadata:
  name: copy-loadbalancer
spec:
  type: LoadBalancer
  ports:
  - port: 10035
    targetPort: 10035
  selector:
    app: copy
```    


### copy-0-lb.yaml

If you wish to access one of the copy instances explicitly so that you can create sessions you can create a load balancer which links to just one instance, in this case the first copy instance which is named "copy-0".

```
apiVersion: v1
kind: Service
metadata:
  name: copy-0-loadbalancer
spec:
  type: LoadBalancer
  ports:
  - port: 10035
    targetPort: 10035
  selector:
    app: copy
    statefulset.kubernetes.io/pod-name: copy-0
```

