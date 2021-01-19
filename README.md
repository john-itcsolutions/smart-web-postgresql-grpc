## INTRODUCTION:

# smart-web-postgresql-grpc
Project based on Arianit Uka's postgres-statefulset merged with Cyber Republic's Elastos Smartweb gRPC-based Service and SQLAlchemy.

To tackle a full Kubernetes installation, ideally you would need a 32 GB RAM; 250 GB SSD; + HDD: PC (x86_64). eg an Extreme Gaming Computer. If you intend to include Machine Learning/AI capabilities, your Kubeflow installation will go much more easily with an 8 core Host rather than a 4 core one.

We base initial development such as this locally.

You need to develop initially on docker. ITCSA uses Ubuntu 20.04 as host platform.

You will not need an Extreme Gaming level of computer for Docker-based (initial - eg. database) work without Kubernetes.

See our website at https://www.itcsolutions.com.au/kubernetes-yaml-file-example/ for an older but more visual idea of this project and others.

## Get:

Docker for Ubuntu: https://docs.docker.com/engine/install/ubuntu/

Remember to

`sudo usermod -aG docker $USER && newgrp docker`

after install is complete.

The docker-based development on this project is adapted from the code in:

https://github.com/cyber-republic/elastos-smartweb-service  (smartweb-server, providing blockchain access, containerised by ITCSA)

and

https://github.com/cyber-republic/python-grpc-adenine  (smartweb-client, database query client with Python)

and

https://github.com/arianitu/postgres-statefulset (replicated postgres database)

The predominant language used to code for this project is Python (here, mainly version 3.8).

_______________________________________________________________________________

## SET UP NODES:

(The database schema for ITCSA's project are private and available only under certain conditions.)

Development is easiest in Docker as opposed to Kubernetes.

Nevertheless, if you wish to go on to build a full set of nodes with High Availability enabled on Kubernetes, we use Multipass/ Microk8s, as follows.

Multipass/Microk8s uses the Docker Container runtime in the current version, however the Kubernetes runtime will change in Kubernetes 1.22 from the Docker runtime to a standardised (Docker-compatible) Container runtime.

On host terminal, preferably in a directory within the 2nd HDD if available, to save working files in case of a crash:

`sudo snap install multipass`

`multipass launch --name primary --mem 2G --disk 20G`

`multipass launch --name database --mem 3G --disk 20G`

`multipass launch --name kubeflow --mem 16G --disk 60G --cpus 4`

Open 3 new terminal tabs.

In first:

`multipass shell primary`

In second:

`multipass shell database`

In third:

`mltipass shell kubeflow`

In first (within primary)

`sudo snap install microk8s --classic --channel=1.19`

.. wait .. when installed:

`sudo iptables -P FORWARD ACCEPT`

`sudo usermod -a -G microk8s $USER`

`sudo chown -f -R $USER ~/.kube`

`sudo passwd ubuntu` (enter new password twice for your new vm default user, "ubuntu")

`su - $USER`

Repeat above 6 commands in each of database and kubeflow nodes.

Check state of microk8s in each node:

`microk8s status --wait-ready` (x3)

if all is well continue below. If any node is no longer running microk8s, try stopping and starting the node. If this also fails,

`sudo snap remove microk8s`

then reinstall as above.

When every node is running microk8s:

On each node:

`microk8s enable dns`

On primary:

`microk8s add-node`

... and copy the join command to the database node.

When the database node has joined, repeat for kubeflow node.

Then (on primary):

`microk8s enable dashboard ingress istio metallb metrics-server registry storage`

.. at the beginning of metallb enablement you need to respond to input request with `192.168.0.0/32`.

Check status of each node with `microk8s status --wait-ready`. If all good, in primary

(Sometimes, the nodes fall over and they may require a restart. After a restart of a machine (virtual or actual) you can try issuing `sudo iptables -P FORWARD ACCEPT && apt-get install iptables-persistent`, followed by a hard reboot of the Host. Reinstalling microk8s, on the node, however is sometimes the only direct way to fix a dead node (kubernetes-wise), unless you are very patient.)

(Note: You need to use `microk8s leave` on any node you intend to stop, delete and purge, as well as issuing `microk8s remove-node <node-name>` on primary.)

Recheck microk8s status on every node.

Next we need to label the nodes. On primary:

`microk8s kubectl label nodes primary nodetype=primary`

`microk8s kubectl label nodes database nodetype=database`

`microk8s kubectl label nodes kubeflow nodetype=kubeflow`

Now in each node we require a shared directory to mount your host repo directory into, and to give general access to files on the Host.

So:

in each node:

`mkdir /home/ubuntu/shared`

then on Host:

`multipass mount /path/to/smart-web-postgresql-grpc primary:/home/ubuntu/shared`

.. and similarly for database and kubeflow nodes.

FROM HOST TERMINAL: clone the repo to your working directory

`git clone https://github.com/john-itcsolutions/smart-web-postgresql-grpc`

`cd smart-web-postgresql-grpc`

______________________________________________________________________________

## Preliminaries:

IN HOST TERMINAL:
 
 Now we pull the images we need:
 
 `docker pull redis:5.0.4`

 `docker pull postgres-10:14`
 
Issue `multipass list` from host and note Ip Address of the primary node.
 
 `sudo nano /etc/docker/daemon.json`

 You need to have something like:
 
`{`

`  insecure-registries : [10.184.36.93:32000,`

`                           ]`

`}`

where the Ip Address must match your primary node address. (Use `multipass list` on the host)

Then:

`sudo systemctl daemon-reload`

`sudo systemctl restart docker`

`docker tag redis:5.0.4 10.184.36.93:32000/redis:5.0.4`

`docker tag postgres-10:14 10.184.36.93:32000/postgres:10.14`

Now push the images to the microk8s registry:

`docker push 10.184.36.93:32000/redis:5.0.4`

`docker push 10.184.36.93:32000/postgres:10.14`

(Remember to change the above Ip Addresses to match your own node address for master node!)

_____________________________________________________________

## DATABASE & REDIS:- Set Up secrets:

Now in database node:

`cd /home/ubuntu/shared/path/to/smart-web-postgresql-grpc/app/postgres-statefulset`

Note: "You can edit secret.yml, and you would have to edit kustomization.yaml, but then you need to alter the redis.yml as the hash for the keys will change. So you would have to find the hashes in that yml file and alter to match newly created keys - from running `microk8s kubectl apply -k .`"

Run `microk8s kubectl apply -f config/secret.yml` and then `cd config && ./create_configmap.sh`    

`cd ../../`

Generate further secrets from kustomization file:

`microk8s kubectl apply -k .`

___________________________________________________________________________________________

 ## Set Up Redis
 
 Ensure you are on database node in the "smart-web-postgresql-grpc/app" directory which you access via the shared folder, and (to be utilised in later development phases)
 
 `sudo ./volumes-redis.sh`
 
 `sudo ./copyredisconf.sh`
 
 which copies redis.conf (unedited as yet) to the config directory.
 
 You then need to edit redis.conf in place (ie in /mnt/disk/config-redis) and insert the name of the data backup folder which the dump.rdb file will be placed in. You must search redis.conf for the correct line to edit.
 
 First see what folders you have:
 
 `ls /mnt/disk`
 
 Then:
 
 `sudo nano /mnt/disk/config-redis/redis.conf`
 
 The name of the data backup foldder is /mnt/disk/data-redis, however, relative to the location of redis.conf you need to insert 
 
 `../data-redis`, inside redis.conf, at the approriate position.
  
 MOVE TO primary TERMINAL:

`cd /home/ubuntu/shared/path/to/smart-web-postgresql-grpc/app`

`microk8s kubectl apply -f redis.yml`

check pods .. `watch microk8s kubectl get pods`

NOTE: As yet Redis is not programmed to act as a Query Cache Server. This requires investment of time and effort in analysis of your DApp's requirements.

________________________________________________________________________________________

## Ingress-NGINX/Redis with MetalLB Load Balancer:

In primary, shared/../path/to/smart-web-postgresql-grpc/app:

`microk8s kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v0.41.2/deploy/static/provider/baremetal/deploy.yaml`

`microk8s kubectl apply -f metallb-configMap.yml`

We need to reconfigure the ingress controller (NGINX) to co-serve the existing "redis" service via a new TCP port:

`microk8s kubectl apply -f reconf-nginx-4-redis.yml`

`microk8s kubectl apply -f ingress-grpc-smart-web.yml`

Check results:

`microk8s kubectl get pods -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx --watch`

By issuing `microk8s kubectl get services --all-namespaces` and `microk8s kubectl get pods --all-namespaces`,

you may view the existing services and pods with associated namespaces. "microk8s kubectl describe .." gives more information.

In particular try:

`microk8s kubectl describe ingress general-ingress`

.. the IP Address and Port shown are the connection details needed by the "manifest.json" file (in your front-end GUI Ionic DApp), in which IP Addresses for the DApp are whitelisted (such as Banks and other Web Services, as well as access to Blockchains and Database).

In order to connect via Redis, we must configure users to connect only to their home database (at this stage, most users would connect to the 'cheirrs' schema). This requires the use of 4 of the 16 available default "databases" in a Redis installation. The 4 Redis databases correspond to the 4 schema in operation: a_horse, cheirrs, cheirrs_oseer & public. The redis databases are labeled redis-database-0 through redis-database-3. Each user must have a "home" database, and be prevented from accessing any other databases as that user.

In the case of network connectivity lost, recommence this section ('Ingress-NGINX/Redis with MetalLB Load Balancer') after doing:

`microk8s disable ingress metallb`

then follow from top of section.

If disabling ingress and metallb fails, then `sudo snap remove microk8s` (in each node), and start from beginning after shelling into the still-existent nodes. Microk8s will take a snapshot (particularly of the postgres master and replica states) and you will find the database already set up for you upon `sudo snap install microk8s --classic --channel=1.19`, etc.

Also remember to `multipass restart <node-name1> [<node-name2> .. [ .. ]]` from Host in times of nonsense.

 _________________________________________________________________
 

## KUBEFLOW and Machine Learning (Artificial Intelligence or Statistical Learning)

On microk8s you will need to have available, and allow for, 16GB of RAM at least, and 4 cpus in your 'kubeflow' virtual machine.

Any compute assistance you can obtain from extra gpu & RAM (NVIDIA standard CUDA graphics cards), is valuable. To enable NVIDIA CUDA support:

`microk8s enable gpu`

##     On kubeflow node:

`microk8s enable kubeflow`

OR, if system reports lower memory to microk8s:

`KUBEFLOW_IGNORE_MIN_MEM=true microk8s.enable kubeflow`

The full enablement process takes MANY minutes on a 32GB 4 core Host. An 8 core machine is actually recommended.

Good luck! (see either https://statlearning.com/ - or -  https://dokumen.pub/introduction-to-statistical-learning-7th-printingnbsped-9781461471370-9781461471387-2013936251.html -  download "An Introduction to Statistical Learning"; Gareth James et al.)

_____________________________________________________________________

## DATABASE
## Create places for Persistent Volumes on database node and allow access:

From postgres-statefulset directory, but in database node:

`sudo ./volumes-postgres.sh`

__________________________________________________________________

## Start master and replica:

In primary node, in "shared" directory and inside "smart-web-postgresql-grpc/app/postgres-statefulset" folder, as above, start master postgres server:

`microk8s kubectl apply -f statefulset-master.yml`

In primary node, check pod:

`watch microk8s kubectl get pods`

If errors or excessive delay get messages with:

`microk8s kubectl describe pods`

 .. fix errors!
 
 In primary node, after master is successfully running and ready, start replica server:

`microk8s kubectl apply -f statefulset-replica.yml`
 
 Check pods in primary node.

__________________________________________________________________________

## Copy sql scripts; Build Database Schema:

From primary, in shared/../smart-web-postgresql-grpc/app/smart-web/grpc_adenine/database/scripts folder:

`microk8s kubectl cp create_table_scripts.sql postgres-0:/var/lib/postgresql/data/`

`microk8s kubectl cp insert_rows_scripts.sql postgres-0:/var/lib/postgresql/data/`

`microk8s kubectl cp reset_database.sql postgres-0:/var/lib/postgresql/data/`

## The following 3 commands would be possible only after you are positively identified, gain our trust, and sign an agreement to work with us, in order to obtain these backup files. Or, develop your own!

`microk8s kubectl cp ../../cheirrs_backup.sql postgres-0:/var/lib/postgresql/data/`

`microk8s kubectl cp ../../cheirrs_oseer_backup.sql postgres-0:/var/lib/postgresql/data/`

`microk8s kubectl cp ../../a_horse_backup.sql postgres-0:/var/lib/postgresql/data/`

exec into master db container:

`microk8s kubectl exec -it postgres-0 -- sh`

Inside postgres-0 container on database node:

`apk add nano`

(To obtain editor. Choose your own if you wish. Note the Postgresql deployment is based on an Alpine Linux image, and is therefore somewhat different to Ubuntu.)

`su postgres`

`createdb general`

`psql general < /var/lib/postgresql/data/cheirrs_backup.sql`

`psql general < /var/lib/postgresql/data/cheirrs_oseer_backup.sql`

`psql general < /var/lib/postgresql/data/a_horse_backup.sql`

Create users in postgres:

`psql general`

`create role cheirrs_user with login password 'passwd';`

`create role cheirrs_admin with superuser login password 'passwd';`

`create role cheirrs_oseer_admin with superuser login password 'passwd';`

`create role a_horse_admin with superuser login password 'passwd';`

`create role gmu with login password 'gmu';`

Check Schemas: there should be 'public, 'a_horse'; 'cheirrs'; 'cheirrs_oseer'.

`\dn`

Check off users:

`\du`

`\dt ` should reveal no instances (in public schema)

`set search_path to 'cheirrs';`

.. now, `\dt` should reveal a full set of 600+ tables in 2 categories: 1) accounting_xyz and 2) uc_uvw (for use_case)

You will see identical table sets at this stage in development, by altering search_path before \dt, to examine cheirrs_oseer and a_horse schemas.

In postgres-0 container on database-node:

Exit psql shell:

`\q`

Run Elastos scripts to prepare database public schema for Blockchains interaction;

`psql -h localhost -d general -U gmu -a -q -f /var/lib/postgresql/data/create_table_scripts.sql`

`psql -h localhost -d general -U gmu -a -q -f /var/lib/postgresql/data/insert_rows_scripts.sql`

Now if you

`psql general`

then

`\dt` (to reveal tables in default public schema) you should see 3 tables.

Try:

`select * from users;`

You should see the single user's details.
____________________________________________________________________________________________

## Blockchains-Database Server (Smart-web) 

Now we turn to setting up the Blockchain/Database gRPC Server Deployment,

In a Host terminal,

`cd path/to/smart-web-postgresql-grpc/app/smart-web`

`docker build -t 10.184.36.93:32000/smart-web:1 .`

.. where the IP Address needs to be appropriate to your installation .. check multipass list for the primary node's address.

`docker push 10.184.36.93:32000/smart-web:1`

Next, move to a primary node terminal:

`cd shared/../smart-web-postgresql-grpc/app/smart-web`

Now, you need to ensure the image tags in the .yml files you are about to build from are in sync with the actual last image tag you built (+1). This comment always applies to smart-web  Docker-built images, as you progress. This means you have to "bump" along the image tags in both the tag given for the Dockerfile build target, and the kubernetes smart-web.yml file that references that image.

`microk8s kubectl apply -f smart-web.yml`

`watch microk8s kubectl get pods`

If errors or excessive delays:

`microk8s kubectl describe pods` will give error messages of sorts.

Also `microk8s kubectl logs deployment/smart-web` can assist.

if successful, or even to set up in a crash loop in order to get IP Addresses to configure postgresql's host-based access, as follows:

`microk8s kubectl exec -it postgres-0 -- sh`

and inside the master postgres container

`nano /var/lib/postgresql/data/pgdata/pg_hba.conf`

making sure to enable the subnet for all the containers on the database node. (redis, smart-web, and when completed, smart-client) with a CIDR of /24. eg 10.1.34.0/24 would represent one subnet where all addresses begin with 10.1.34. Since all communicating containers are together on "database", we only need to specify a single subnet by looking up pod details with

`microk8s kubectl describe pods`

and noting the three common initial fields of the IP Addresses of the pods in the database node. The subnet is constucted as field-1.field-2.field-3.0/24

_____________________________________________________________

## TESTING



# To be continued ..
