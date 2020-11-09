> TODO : Rollback procedure using restore

# List of nodes

Check the list in the corresponding file <cluster>.md

# Reference documentation

https://docs.mirantis.com/docker-enterprise/v3.0/dockeree-products/ucp/admin/disaster-recovery.html

# Chronogram

| Action        | Duration           |
| ------------- |:-------------:|
| [0. Open a proactive support case](#0-open-a-proactive-support-case) | 5 minutes |
| [1. Take a support dump](#1-take-a-support-dump) | 5 minutes |
| [2. Check available backups](#2-check-available-backups) | 15 minutes |
| [3. Check current health of the cluster](#3-check-current-health-of-the-cluster) | 15 minutes |
| [:arrow_forward: **CHECKPOINT** Start of DR](#checkpoint-start-of-dr) |  |
| [4. Communicate start of DR](#4-communicate-start-of-DR) | 5 minutes |
| [5. Stop VMs in Datacenter dc1](#5-stop-vms-in-datacenter-dc1) | 15 minutes |
| [:arrow_forward: **CHECKPOINT** No rollback possible beyond this point](#checkpoint-no-rollback-possible-beyond-this-point) |  |
| [6. Restore UCP quorum](#6-restore-ucp-quorum) | 40 minutes |
| [7. Restore DTR quorum](#7-restore-dtr-quorum) | 10 minutes |
| [:arrow_forward: **CHECKPOINT** UCP and DTR are UP on dc2](#checkpoint-ucp-and-dtr-are-up-on-dc2) |  |
| [8. Communicate end of failover to dc2](#8-communicate-end-of-failover-to-dc2) | 5 minutes |
| [9. Applications testings](#9-applications-testings) | 120 minutes |
| [:arrow_forward: **CHECKPOINT** Adding back dc1 nodes to the cluster](#checkpoint-adding-back-marcoussis-nodes-to-the-cluster) |  |
| [10. Add workers](#10-add-workers) | 10 minutes |
| [11. Add DTR replicas](#11-add-dtr-replicas) | 30 minutes |
| [12. Add managers](#12-add-managers) | 30 minutes |
| [13. Check current health of the cluster](#13-check-cluster-health) | 15 minutes |
| [:arrow_forward: **CHECKPOINT** DR complete](#checkpoint-dr-complete) |  |


# 0. Open a proactive support case

[Back to the chronogram](#chronogram)

# 1. Take a support dump

https://docs.mirantis.com/docker-enterprise/v3.0/dockeree-products/get-support.html#use-the-web-ui-to-get-a-support-dump

Before starting the procedure, connect to UCP and take a full support dump.

[Back to the chronogram](#chronogram)

# 2. Check available backups

:warning: The recovery of the quorum is a destructive process. In case anything goes wrong, there is no way to rollback to the previous state, and you will need to restore the cluster to its initial state by restoring a backup.

https://docs.mirantis.com/docker-enterprise/v3.0/dockeree-products/dee-intro/manage/backup.html

## Check Swarm Backup

https://docs.mirantis.com/docker-enterprise/v3.0/dockeree-products/ucp/admin/disaster-recovery/backup-swarm.html#backup-swarm

## Check UCP Backup

https://docs.mirantis.com/docker-enterprise/v3.0/dockeree-products/ucp/admin/disaster-recovery/backup-ucp.html

If you need to take a backup, connect to *manager3* (the manager on dc2) :  

```shell
UCP_DOCKERHUB=$(docker ps --filter "name=ucp-proxy" --format "{{.Image}}" | cut -d"/" -f1)
UCP_VERSION=$(docker ps --filter "name=ucp-proxy" --format "{{.Image}}" | cut -d":" -f2)
UCP_IMAGE=${UCP_DOCKERHUB}/ucp:${UCP_VERSION}

docker container run \
    --rm \
    --security-opt label=disable \
    --log-driver none \
    --name ucp \
    --volume /var/run/docker.sock:/var/run/docker.sock \
    --volume /tmp:/backup \
    ${UCP_IMAGE} backup \
    --debug \
    --file ucp-backup-$(date +%Y%m%d-%H_%M_%S).tar \
    --no-passphrase

```

:warning: This backup is not encrypted and contains some sensitive informations. Follow the documentation to encrypt the backup.

## Check DTR Backup

https://docs.mirantis.com/docker-enterprise/v3.0/dockeree-products/dtr/dtr-admin/disaster-recovery/create-a-backup.html

If you need to take a backup, connect to *dtr3* (the DTR replica on dc2) :

```shell
DTR_DOCKERHUB=$(docker ps --filter "name=dtr-registry" --format "{{.Image}}" | cut -d"/" -f1)
DTR_VERSION=$(docker ps --filter "name=dtr-registry" --format "{{.Image}}" | cut -d":" -f2)
DTR_IMAGE=${DTR_DOCKERHUB}/dtr:${DTR_VERSION}

UCP_URL=

read -p 'ucp-admin: ' UCP_ADMIN; \
read -sp 'ucp-password: ' UCP_PASSWORD; \
REPLICA_ID=$(docker ps --format '{{.Names}}' -f name=dtr-rethink | cut -f 3 -d '-'); \
docker run --log-driver none -i --rm \
  --env UCP_PASSWORD=$UCP_PASSWORD \
  $DTR_IMAGE backup \
  --ucp-username $UCP_ADMIN \
  --ucp-url $UCP_URL \
  --ucp-insecure-tls \
  --existing-replica-id $REPLICA_ID > dtr-metadata-backup-$(date +%Y%m%d-%H_%M_%S).tar

```

:warning: This backup is not encrypted and contains some sensitive informations. Follow the documentation to encrypt the backup.

[Back to the chronogram](#chronogram)

# 3. Check current health of the cluster

Before starting the operations, check that the cluster is healthy.

## Check UCP health

- Connect to the UCP UI and check that there is no banner with error or warning.

- Check the UCP API

```shell
UCP_URL=

curl -ks https://${UCP_URL}/_ping

```

- Using a client bundle, check that every nodes are OK

```shell
cd <BUNDLE_DIR>
source env.sh

docker node ls
kubectl get nodes

```

Every nodes should be in *Ready* state

## Check DTR health

- Connect to the DTR UI and check that there is no banner with error or warning.

- Check the DTR API

```shell
DTR_URL=

read -p 'ucp-admin: ' UCP_ADMIN; \
read -sp 'ucp-password: ' UCP_PASSWORD; \
curl -ksL -u $UCP_ADMIN:$UCP_PASSWORD https://${DTR_URL}/api/v0/meta/cluster_status | jq .replica_health

```

## Check Deployed apps health

A test application has been deployed in the cluster.  
You can check the availability of the application  


[Back to the chronogram](#chronogram)

# CHECKPOINT Start of DR

[Back to the chronogram](#chronogram)

# 4. Communicate start of DR

[Back to the chronogram](#chronogram)

# 5. Stop VMs in Datacenter dc1

> NOTE : For the DR exercise, you will just stop the docker engine.  

Stop the docker engine on the nodes in the table [List of Nodes](#list-of-nodes) that are in Datacenter dc1.  

First stop managers, then dtrs, then workers in order to loose quorum before Kubernetes is able to reschedule Pods on dc2.  

After the shutdown :

- UCP UI is down
- docker bundle and kubectl are KO

```shell
kubectl get pods

Error from server (InternalError): an error on the server ("unknown") has prevented the request from succeeding (get pods)

```

- *manager3* (remaining manager on dc2) has lost quorum

Execute these commands on *manager3* (the remaining manager on dc2) : 

```shell
docker node ls

Error response from daemon: rpc error: code = Unknown desc = The swarm does not have a leader. It's possible that too few managers are online. Make sure more than half of the managers are online.

```

- DTR is down

```shell
docker login <dtr>

Username (admin): 
Password:

Error response from daemon: Get https://dtr/v2/: received unexpected HTTP status: 500 Internal Server Error

```

- Replicas of the apps running on dc1 are UP  


[Back to the chronogram](#chronogram)

# CHECKPOINT No rollback possible beyond this point

Until this point, rollback is as simple as starting again all the VMs on dc1.  
After executing the following steps, the rollback procedure if anything goes wrong will be to restore the cluster with a backup.  

[Back to the chronogram](#chronogram)

# 6. Restore UCP quorum

https://docs.mirantis.com/docker-enterprise/v3.0/dockeree-products/ucp/admin/monitor-and-troubleshoot.html

https://docs.mirantis.com/docker-enterprise/v3.0/dockeree-products/ucp/admin/disaster-recovery.html

## Restore Swarm quorum

Force a new Swarm cluster, and force a new Swarm snapshot to happen.  

Execute these commands on *manager3* (the remaining manager on dc2) :

```shell
docker swarm init --force-new-cluster --default-addr-pool 10.10.0.0/16 --default-addr-pool 10.20.0.0/16 --default-addr-pool-mask-length 21
docker swarm update --snapshot-interval 1
docker network create temporarynet -d overlay
docker network rm temporarynet
docker swarm update --snapshot-interval 10000

```

Wait for workers to become Ready, stopped VMs should appear as Down

```shell
watch docker node ls

```


Now Swarm quorum is restored and Swarm commands are working on *manager3* :

```shell
docker node ls
docker service ls

```

## Restore etcd quorum

Execute these commands on *manager3* (the remaining manager on dc2) :

```shell
UCP_VERSION=$(docker ps --filter "name=ucp-proxy" --format "{{.Image}}" | cut -d":" -f2)
UCP_DOCKERHUB=$(docker ps --filter "name=ucp-proxy" --format "{{.Image}}" | cut -d"/" -f1)
UCP_ETCD_IMAGE=${UCP_DOCKERHUB}/ucp-etcd:${UCP_VERSION}

NODE_NAME=$(docker info --format '{{.Name}}')

docker node update --label-add com.docker.ucp.agent-pause=true $NODE_NAME
docker stop ucp-kv

```

Move the following file to another location if it exists :

```shell
mv /var/lib/docker/volumes/ucp-kv/_data/datav3/extra.args /tmp

```

Create a temporary etcd container :

```shell
docker run --rm -it --entrypoint sh -v ucp-kv:/data -v ucp-kv-certs:/etc/docker/ssl $UCP_ETCD_IMAGE

```

> NOTE : If you get an error **docker: Error response from daemon: OCI runtime create failed**, do the following actions.  
> Then, re-execute the command to create a temporary etcd container.
```shell
systemctl restart docker
# wait a few seconds
docker stop ucp-kv
# check that ucp-kv is not started
docker ps | grep ucp-kv

```


When the temporary etcd container is launched, run this command inside the container

```shell
/bin/etcd --data-dir /data/datav3 --force-new-cluster

```

Wait for the message **ready to service client requests**, then exit the container using <CONTROL+C> <CONTROL+V>

```shell
docker start ucp-kv
docker node update --label-rm com.docker.ucp.agent-pause $NODE_NAME

```

Wait a few seconds, then check etcd Health

```shell
docker exec -it ucp-kv etcdctl --endpoint https://127.0.0.1:2379 --ca-file /etc/docker/ssl/ca.pem --cert-file /etc/docker/ssl/cert.pem --key-file /etc/docker/ssl/key.pem cluster-health 

docker exec -it ucp-kv etcdctl --endpoint https://127.0.0.1:2379 --ca-file /etc/docker/ssl/ca.pem --cert-file /etc/docker/ssl/cert.pem --key-file /etc/docker/ssl/key.pem member list 

```

Etcd cluster has now only 1 replica

## Restore rethinkdb quorum

- Restore quorum  

Execute these commands on *manager3* (the remaining manager on dc2) :

```shell
UCP_VERSION=$(docker ps --filter "name=ucp-proxy" --format "{{.Image}}" | cut -d":" -f2)
UCP_DOCKERHUB=$(docker ps --filter "name=ucp-proxy" --format "{{.Image}}" | cut -d"/" -f1)
UCP_AUTH_IMAGE=${UCP_DOCKERHUB}/ucp-auth:${UCP_VERSION}
NUM_MANAGERS=1
NODE_ADDRESS=$(docker info --format '{{.Swarm.NodeAddr}}')

docker container run \
    --rm -v ucp-auth-store-certs:/tls ${UCP_AUTH_IMAGE} \
    --db-addr=${NODE_ADDRESS}:12383 \
    --debug reconfigure-db \
    --num-replicas ${NUM_MANAGERS} \
    --emergency-repair

```

> NOTE : Restart the container *ucp-auth-store* if you get the following error, then run again the **emergency-repair** command :

```shell
time="2020-11-03T15:01:00Z" level=debug msg="Connecting to db ..."
time="2020-11-03T15:01:00Z" level=debug msg="connecting to DB Addrs: [172.0.0.1:12383]"
time="2020-11-03T15:01:00Z" level=fatal msg="unable to connect to database cluster: rethinkdb: read tcp 172.0.0.1:48350->172.0.0.1:12383: read: connection reset by peer"

```


- Check rethinkdb health

```shell
UCP_VERSION=$(docker ps --filter "name=ucp-proxy" --format "{{.Image}}" | cut -d":" -f2)
UCP_DOCKERHUB=$(docker ps --filter "name=ucp-proxy" --format "{{.Image}}" | cut -d"/" -f1)
UCP_AUTH_IMAGE=${UCP_DOCKERHUB}/ucp-auth:${UCP_VERSION}
NODE_ADDRESS=$(docker info --format '{{.Swarm.NodeAddr}}')

docker container run \
    --rm -v ucp-auth-store-certs:/tls ${UCP_AUTH_IMAGE} \
    --db-addr=${NODE_ADDRESS}:12383 \
    --debug db-status

```

- Restart docker engine

```shell
systemctl restart docker

```

## Check UCP health

- Connect to the UCP UI and check that there is no banner with error or warning.

- Check the UCP API

```shell
UCP_URL=

curl -ks https://${UCP_URL}/_ping

```

- Using a client bundle, check that UCP and Kubernetes responds

```shell
cd <BUNDLE_DIR>
source env.sh

docker node ls
kubectl get nodes
kubectl get pods -A

```

> Note : 
> - nginx-controller replicas of **dc1** will stay *Pending* because of PodAntiaffinity on Datacenter.  
> - Apps replicas od **dc1** will be scheduled on workers of **dc2** after 5 minutes, but can be stuck in ContainerCreating state if image is not present locally and DTR is not available.

## Remove managers on dc1 from the cluster

Now that docker bundle is working again, using the docker bundle, remove old managers on dc1 (not dtr and workers)

```shell
cd <BUNDLE_DIR>
source env.sh

docker node rm <manager1> <manager2>

```

[Back to the chronogram](#chronogram)

# 7. Restore DTR quorum

https://docs.mirantis.com/docker-enterprise/v3.0/dockeree-products/dtr/dtr-admin/monitor-and-troubleshoot.html

https://docs.mirantis.com/docker-enterprise/v3.0/dockeree-products/dtr/dtr-admin/disaster-recovery.html

## Restore rethinkdb quorum

Execute these commands on *dtr3* (DTR replica in dc2) : 

```shell
DTR_DOCKERHUB=$(docker ps --filter "name=dtr-registry" --format "{{.Image}}" | cut -d"/" -f1)
DTR_VERSION=$(docker ps --filter "name=dtr-registry" --format "{{.Image}}" | cut -d":" -f2)
DTR_IMAGE=${DTR_DOCKERHUB}/dtr:${DTR_VERSION}

UCP_URL=

REPLICA_ID=$(docker inspect -f '{{.Name}}' $(docker ps -q -f name=dtr-rethink) | cut -f 3 -d '-')
read -p 'ucp-admin: ' UCP_ADMIN; \
docker run -it --rm ${DTR_IMAGE} emergency-repair \
  --ucp-url ${UCP_URL} \
  --ucp-username ${UCP_ADMIN} \
  --ucp-insecure-tls \
  --existing-replica-id $REPLICA_ID

```

## Check DTR health

- Check that there is now 1 replica available

```shell
DTR_URL=

read -p 'ucp-admin: ' UCP_ADMIN ; \
read -sp 'ucp-password: ' UCP_PASSWORD ; \
curl -ksL -u $UCP_ADMIN:$UCP_PASSWORD https://${DTR_URL}/api/v0/meta/cluster_status | jq .replica_health

```

Test that you are able to pull an image

```shell
docker image pull <IMAGE ON DTR>

```

[Back to the chronogram](#chronogram)

# CHECKPOINT UCP and DTR are UP on dc2

[Back to the chronogram](#chronogram)

# 8. Communicate end of failover to dc2

[Back to the chronogram](#chronogram)

# 9. Applications testings

[Back to the chronogram](#chronogram)

# CHECKPOINT Adding back dc1 nodes to the cluster

[Back to the chronogram](#chronogram)

# 10. Add workers

https://docs.mirantis.com/docker-enterprise/v3.0/dockeree-products/ucp/admin/configure/join-nodes/join-linux-nodes-to-cluster.html

- To add back the worker nodes to the cluster, just start the VMs and they will automatically join the cluster.  

- Check the status of the nodes using a client bundle:

```shell
cd <BUNDLE DIR>
source env.sh

kubectl get nodes
docker node ls

```

> NOTE : Sometimes the nodes will take some time to appear as Ready.  
> You may need to restart ucp-swarm-manager container on the manager to update swarm classic inventory
> Execute these commands on *manager3* (remaining manager in dc2) : 

```shell
docker restart ucp-swarm-manager

```

[Back to the chronogram](#chronogram)

# 11. Add DTR replicas

https://docs.mirantis.com/docker-enterprise/v3.0/dockeree-products/dtr/dtr-admin/configure/set-up-high-availability.html

- Follow the same procedure than for the workers to add the nodes back to the Swarm cluster.

- Check Allow administrators to deploy containers on UCP managers or nodes running DTR

- Before adding the DTR replica to the DTR cluster, clear old DTR state on each dtr node on **dc1**

On each dtr node on **dc1** :

```shell
docker container rm -f $(docker ps -aq --filter "name=dtr-")
docker volume rm $(docker volume ls -q --filter "name=dtr-")

```

- Join new replicas to cluster

Execute these commands on *dtr3* (the DTR replica on dc2) :  

```shell
DTR_DOCKERHUB=$(docker ps --filter "name=dtr-registry" --format "{{.Image}}" | cut -d"/" -f1)
DTR_VERSION=$(docker ps --filter "name=dtr-registry" --format "{{.Image}}" | cut -d":" -f2)
DTR_IMAGE=${DTR_DOCKERHUB}/dtr:${DTR_VERSION}

UCP_URL=

read -p 'ucp-admin: ' UCP_ADMIN

# Replace <dtrX> with tne name of the node you want to add
docker run -it --rm ${DTR_IMAGE} join --ucp-node <dtr1> --ucp-insecure-tls --ucp-url $UCP_URL --ucp-username $UCP_ADMIN
# Wait for the replica to join the cluster, then join the last replica
docker run -it --rm ${DTR_IMAGE} join --ucp-node <dtr2> --ucp-insecure-tls --ucp-url $UCP_URL --ucp-username $UCP_ADMIN

```


[Back to the chronogram](#chronogram)

# 12. Add managers

https://docs.mirantis.com/docker-enterprise/v3.0/dockeree-products/ucp/admin/configure/join-nodes/set-up-high-availability.html

> TODO : check labels

:warning:  :warning: :warning: **It is recommended to wipe the old managers on dc1 and to replace them with 2 new nodes.**  

For the exercise, you will reuse old nodes *manager1* and *manager2*.  

:warning: Do not start the two old nodes at the same time, because it can create a split brain issue with etcd and rethinkdb.  

First start *manager1* and wipe datas on it before starting *manager2*  

- Wipe the datas in order to prevent it to reconnect automatically to the old etcd/rethinkdb clusters

Execute the following commands on *manager1*, and then on *manager2*

```shell
systemctl stop docker

# Wait a few seconds then wipe datas
rm -rf /var/lib/docker/*
systemctl start docker

```

> NOTE : you can discard the following error :

```shell
rm: cannot remove '/var/lib/docker/containers': Device or resource busy
rm: cannot remove '/var/lib/docker/volumes': Device or resource busy 
```

- Load UCP images

```shell
UCP_VERSION=3.2.4
curl https://<images>/ucp_images_${UCP_VERSION}.tar.gz | docker load

```

- Join the first node as a manager

Execute the following command on *manager3*  (remaining node in dc2) in order to generate a join token

```shell
docker swarm join-token manager

```

Paste the command on the node you want to join

```shell
docker swarm join <TOKEN HOST:IP>

```

Wait until the manager has fully joined the cluster and is healthy, then repeat operation for the second manager

> NOTE : the command can timeout, it's normal. It can takes up to 5 minutes to join the cluster.  

[Back to the chronogram](#chronogram)

# 13. Check cluster health

Follow the same procedure as before in the chapter [3. Check current health of the cluster](#3-check-current-health-of-the-cluster)

[Back to the chronogram](#chronogram)

# CHECKPOINT DR complete

# TODO : Restore Backup

```shell

# manager3
docker container run --rm -it --security-opt label=disable -v /var/run/docker.sock:/var/run/docker.sock --name ucp $UCP_IMAGE uninstall-ucp --interactive
docker container run --rm -i --security-opt label=disable -v /var/run/docker.sock:/var/run/docker.sock --name ucp $UCP_IMAGE restore < /tmp/mybackup.tar
docker node rm <manager1> <manager2>

# dtr3
docker run -it --rm mirantis/dtr:2.7.8 destroy --ucp-insecure-tls

DTR_IMAGE=mirantis/dtr:2.7.8

UCP_URL=

# TODO : docker run -it --rm ${DTR_IMAGE} restore --ucp-node $(hostname) --ucp-insecure-tls --ucp-url $UCP_URL --ucp-username $USER < dtr-metadata-backup-20201030-15_59_32.tar

```
