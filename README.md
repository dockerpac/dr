# TEST PREPARATION

# List of nodes
| NODE        | HOSTNAME           | DATACENTER |
|:-------------:|:-------------:|:-------------:|
| *manager1*| ip-172-31-2-202.eu-central-1.compute.internal | dc1
| *manager2*| ip-172-31-25-32.eu-central-1.compute.internal | dc1
| *manager3*| ip-172-31-43-229.eu-central-1.compute.internal | dc2
| *dtr1* | ip-172-31-11-254.eu-central-1.compute.internal | dc1
| *dtr2* | ip-172-31-23-173.eu-central-1.compute.internal | dc1
| *dtr3* | ip-172-31-44-29.eu-central-1.compute.internal | dc2
| *worker1* | ip-172-31-10-46.eu-central-1.compute.internal  | dc1
| *worker2* | ip-172-31-18-139.eu-central-1.compute.internal  | dc1
| *worker3* | ip-172-31-33-35.eu-central-1.compute.internal  | dc2
| *worker4* | ip-172-31-39-74.eu-central-1.compute.internal | dc2

# TODO
```shell
# ip-172-31-2-202.eu-central-1.compute.internal (dc1) : 3.123.25.148
# ip-172-31-25-32.eu-central-1.compute.internal (dc1) : 3.127.243.103
# ip-172-31-43-229.eu-central-1.compute.internal (dc2) : 3.125.120.162

# ip-172-31-11-254.eu-central-1.compute.internal (dc1) : 18.195.95.234
# ip-172-31-23-173.eu-central-1.compute.internal (dc1) : 3.122.236.155
# ip-172-31-44-29.eu-central-1.compute.internal (dc2) : 18.192.198.110

# ip-172-31-10-46.eu-central-1.compute.internal (dc1) : 18.197.96.190
# ip-172-31-18-139.eu-central-1.compute.internal (dc1) : 18.194.241.2
# ip-172-31-33-35.eu-central-1.compute.internal (dc2) : 3.121.232.153
# ip-172-31-39-74.eu-central-1.compute.internal (dc2) : 18.192.206.206


# Label nodes

k label node ip-172-31-10-46.eu-central-1.compute.internal \
	     ip-172-31-11-254.eu-central-1.compute.internal \
             ip-172-31-18-139.eu-central-1.compute.internal \
             ip-172-31-2-202.eu-central-1.compute.internal \
             ip-172-31-23-173.eu-central-1.compute.internal \
             ip-172-31-25-32.eu-central-1.compute.internal \
        dc=dc1 \
        node-role.kubernetes.io/dc1=

k label node ip-172-31-33-35.eu-central-1.compute.internal \
	     ip-172-31-39-74.eu-central-1.compute.internal \
             ip-172-31-43-229.eu-central-1.compute.internal \
             ip-172-31-44-29.eu-central-1.compute.internal \
        dc=dc2 \
        node-role.kubernetes.io/dc2=

k label node ip-172-31-11-254.eu-central-1.compute.internal \
             ip-172-31-23-173.eu-central-1.compute.internal \
             ip-172-31-44-29.eu-central-1.compute.internal \
        node-role.kubernetes.io/dtr=

k label node ip-172-31-10-46.eu-central-1.compute.internal \
             ip-172-31-18-139.eu-central-1.compute.internal \
             ip-172-31-33-35.eu-central-1.compute.internal \
             ip-172-31-39-74.eu-central-1.compute.internal \
        node-role.kubernetes.io/worker=

k label node ip-172-31-10-46.eu-central-1.compute.internal \
             ip-172-31-33-35.eu-central-1.compute.internal \
        node-role.kubernetes.io/ingress=
```

TODO : Lancer une application de test type dockerdemo ??

# Reference documentation

https://docs.mirantis.com/docker-enterprise/v3.0/dockeree-products/ucp/admin/disaster-recovery.html

# Chronogram

| Action        | Duration           |
| ------------- |:-------------:|
| [Take a support dump](#take-a-support-dump) | 5 minutes |
| [Check available backups](#check-available-backups) | 15 minutes |
| [Check current health of the cluster](#check-current-health-of-the-cluster) | 15 minutes |
| [Communicate start of DR](#communicate-start-of-DR) | 5 minutes |
| [Stop VMs in Datacenter Marcoussis](#stop-vms-in-datacenter-marcoussis) | 15 minutes |
| [CHECKPOINT No rollback possible beyond this point](#checkpoint-no-rollback-possible-beyond-this-point) |  |
| [Restore UCP quorum](#restore-ucp-quorum) | 10 minutes |
| [Restore DTR quorum](#restore-dtr-quorum) | 10 minutes |
| [CHECKPOINT UCP and DTR are UP on Tigery](#checkpoint-ucp-and-dtr-are-up-on-tigery) |  |
| [Communicate end of failover to Tigery](#communicate-end-of-failover-to-tigery) | 5 minutes |
| [Applications testings](#applications-testings) | 120 minutes |
| [CHECKPOINT Adding back Marcoussis nodes to the cluster](#checkpoint-adding-back-marcoussis-nodes-to-the-cluster) |  |
| [Add workers](#add-workers) | 10 minutes |
| [Add DTR replicas](#add-dtr-replicas) | 30 minutes |
| [Add managers](#add-managers) | 20 minutes |
| [Check current health of the cluster](#check-cluster-health) | 15 minutes |
| [CHECKPOINT DR complete](#checkpoint-dr-complete) |  |


TODO : ROLLBACK PROCEDURE USING RESTORE


# Take a support dump

https://docs.mirantis.com/docker-enterprise/v3.0/dockeree-products/get-support.html#use-the-web-ui-to-get-a-support-dump

Before starting the procedure, connect to UCP and take a full support dump.

[Back to the chronogram](#chronogram)

# Check available backups

The recovery of the quorum is a destructive process. In case anything goes wrong, there is no way to rollback to the previous state, and you will need to restore the cluster to its initial state by restoring a backup.

https://docs.mirantis.com/docker-enterprise/v3.0/dockeree-products/dee-intro/manage/backup.html

## Check Swarm Backup
https://docs.mirantis.com/docker-enterprise/v3.0/dockeree-products/ucp/admin/disaster-recovery/backup-swarm.html#backup-swarm

## Check UCP Backup
https://docs.mirantis.com/docker-enterprise/v3.0/dockeree-products/ucp/admin/disaster-recovery/backup-ucp.html

If you need to take a backup, connect to the manager on Tigery : 
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
    --file mybackup.tar \
    --no-passphrase
```

## Check DTR Backup
https://docs.mirantis.com/docker-enterprise/v3.0/dockeree-products/dtr/dtr-admin/disaster-recovery/create-a-backup.html

If you need to take a backup, connect to the DTR replica on Tigery :

```shell
DTR_DOCKERHUB=$(docker ps --filter "name=dtr-registry" --format "{{.Image}}" | cut -d"/" -f1)
DTR_VERSION=$(docker ps --filter "name=dtr-registry" --format "{{.Image}}" | cut -d":" -f2)
DTR_IMAGE=${DTR_DOCKERHUB}/dtr:${DTR_VERSION}

UCP_URL=ucp.pac-amberjack.dockerps.io
UCP_ADMIN=totoadmin

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

[Back to the chronogram](#chronogram)

# Check current health of the cluster

Before starting the operations, check that the cluster is healthy.

## Check UCP health

- Connect to the UCP UI and check that there is no banner with error or warning.

- Check the UCP API

```shell
UCP_URL=ucp.pac-amberjack.dockerps.io

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
DTR_URL=dtr.pac-amberjack.dockerps.io
UCP_ADMIN=totoadmin

read -sp 'ucp-password: ' UCP_PASSWORD; curl -ksL -u $UCP_ADMIN:$UCP_PASSWORD https://${DTR_URL}/api/v0/meta/cluster_status | jq .replica_health
```

## Check Deployed apps health
TODO : check dockerdemo ?

[Back to the chronogram](#chronogram)

# Communicate start of DR

[Back to the chronogram](#chronogram)

# Stop VMs in Datacenter Marcoussis
Stop vms in the table [List of Nodes](#list-of-nodes) that are in Datacenter Marcoussis

After the shutdown :
- UCP UI is down
- docker bundle and kubectl are KO
```shell
kubectl get pods

Error from server (InternalError): an error on the server ("unknown") has prevented the request from succeeding (get pods)
```

- remaining manager has lost quorum

Connect on the remaining manager :
```shell
docker node ls

Error response from daemon: rpc error: code = Unknown desc = The swarm does not have a leader. It's possible that too few managers are online. Make sure more than half of the managers are online.
```

- DTR is down
```shell
docker login dtr.pac-amberjack.dockerps.io

Username (admin): admin
Password:

Error response from daemon: Get https://dtr.pac-amberjack.dockerps.io/v2/: received unexpected HTTP status: 500 Internal Server Error
```

- Replicas of the apps running on Marcoussis are UP

TODO : check dockerdemo health ?

[Back to the chronogram](#chronogram)

# CHECKPOINT No rollback possible beyond this point

[Back to the chronogram](#chronogram)

# Restore UCP quorum
https://docs.mirantis.com/docker-enterprise/v3.0/dockeree-products/ucp/admin/monitor-and-troubleshoot.html

https://docs.mirantis.com/docker-enterprise/v3.0/dockeree-products/ucp/admin/disaster-recovery.html

## Restore Swarm quorum
Force a new Swarm cluster, and force a new Swarm snapshot to happen.  

Execute these commands on the remaining manager on Tigery :
```shell
# TODO : Check existing parameters ???
docker swarm init --force-new-cluster
docker swarm update --snapshot-interval 1
docker network create temporarynet -d overlay
docker network rm temporarynet
docker swarm update --snapshot-interval 10000
```


Wait for workers to become Ready, stopped VMs should appear as Down
```shell
docker node ls
```

Remove old managers (not dtr and workers)
```shell
docker node rm <manager1> <manager2>
```

Now Swarm quorum is restored and Swarm commands are working :
```shell
docker node ls
docker service ls
```

## Restore etcd quorum
Execute these commands on the remaining manager on Tigery :
```shell
UCP_VERSION=$(docker ps --filter "name=ucp-proxy" --format "{{.Image}}" | cut -d":" -f2)
UCP_DOCKERHUB=$(docker ps --filter "name=ucp-proxy" --format "{{.Image}}" | cut -d"/" -f1)
UCP_ETCD_IMAGE=${UCP_DOCKERHUB}/ucp-etcd:${UCP_VERSION}

NODE_NAME=$(docker info --format '{{.Name}}')

docker node update --label-add com.docker.ucp.agent-pause=true $NODE_NAME
docker stop ucp-kv
docker run --rm -it --entrypoint sh -v ucp-kv:/data -v ucp-kv-certs:/etc/docker/ssl $UCP_ETCD_IMAGE
```

Then run this command inside the container

```shell
/bin/etcd --data-dir /data/datav3 --force-new-cluster
```

Wait for the message "ready to service client requests", then exit container using <CONTROL+C> <CONTROL+V>

```shell
docker start ucp-kv
docker node update --label-rm com.docker.ucp.agent-pause $NODE_NAME
```

Check etcd Health

```shell
docker exec -it ucp-kv etcdctl --endpoint https://127.0.0.1:2379 --ca-file /etc/docker/ssl/ca.pem --cert-file /etc/docker/ssl/cert.pem --key-file /etc/docker/ssl/key.pem cluster-health 

docker exec -it ucp-kv etcdctl --endpoint https://127.0.0.1:2379 --ca-file /etc/docker/ssl/ca.pem --cert-file /etc/docker/ssl/cert.pem --key-file /etc/docker/ssl/key.pem member list 
```

Etcd cluster has now only 1 replica

## Restore rethinkdb quorum

- Restore quorum  

Execute these commands on the remaining manager on Tigery :
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
UCP_URL=ucp.pac-amberjack.dockerps.io

curl -ks https://${UCP_URL}/_ping
```

- Using a client bundle, check that UCP and Kubernetes responds

```shell
cd <BUNDLE_DIR>
source env.sh

docker node ls
kubectl get nodes
```


TODO :  
APPS will be rescheduled after 5 minutes.  
Nginx replica will stay Pending because of PodAntiaffinity on Datacenter.  
Apps replicas will be scheduled on workers of DC2 but can be stuck in ContainerCreating state if image is not present locally and DTR is not available.

[Back to the chronogram](#chronogram)

# Restore DTR quorum

https://docs.mirantis.com/docker-enterprise/v3.0/dockeree-products/dtr/dtr-admin/monitor-and-troubleshoot.html

https://docs.mirantis.com/docker-enterprise/v3.0/dockeree-products/dtr/dtr-admin/disaster-recovery.html

Connect on remaining DTR node in Tigery

## Restore rethinkdb quorum

```shell
DTR_DOCKERHUB=$(docker ps --filter "name=dtr-registry" --format "{{.Image}}" | cut -d"/" -f1)
DTR_VERSION=$(docker ps --filter "name=dtr-registry" --format "{{.Image}}" | cut -d":" -f2)
DTR_IMAGE=${DTR_DOCKERHUB}/dtr:${DTR_VERSION}

UCP_URL=ucp.pac-amberjack.dockerps.io
USER=totoadmin

REPLICA_ID=$(docker inspect -f '{{.Name}}' $(docker ps -q -f name=dtr-rethink) | cut -f 3 -d '-')
docker run -it --rm ${DTR_IMAGE} emergency-repair \
  --ucp-url ${UCP_URL} \
  --ucp-username ${USER} \
  --ucp-insecure-tls \
  --existing-replica-id $REPLICA_ID
  ```

## Check DTR health

- Check that there is now 1 replica available
```shell
DTR_URL=dtr.pac-amberjack.dockerps.io
UCP_ADMIN=totoadmin

read -sp 'ucp-password: ' UCP_PASSWORD ; curl -ksL -u $UCP_ADMIN:$UCP_PASSWORD https://${DTR_URL}/api/v0/meta/cluster_status | jq .replica_health
```

Test that you are able to pull an image
```shell
docker image pull <IMAGE ON DTR>
```

[Back to the chronogram](#chronogram)

# CHECKPOINT UCP and DTR are UP on Tigery

[Back to the chronogram](#chronogram)

# Communicate end of failover to Tigery

[Back to the chronogram](#chronogram)

# Applications testings

[Back to the chronogram](#chronogram)

# CHECKPOINT Adding back Marcoussis nodes to the cluster

[Back to the chronogram](#chronogram)

# Add workers
https://docs.mirantis.com/docker-enterprise/v3.0/dockeree-products/ucp/admin/configure/join-nodes/join-linux-nodes-to-cluster.html

To add back the worker nodes to the cluster, just start the VMs and they will automatically join the cluster.  
Check the status of the nodes using a client bundle:
```shell
cd <BUNDLE DIR>
source env.sh

kubectl get nodes
docker node ls
```
Sometimes the nodes will take some time to appear as Ready.  
You may need to restart ucp-swarm-manager container on the manager to update swarm classic inventory

On the remaining manager :
```shell
docker restart ucp-swarm-manager
```

[Back to the chronogram](#chronogram)

# Add DTR replicas

https://docs.mirantis.com/docker-enterprise/v3.0/dockeree-products/dtr/dtr-admin/configure/set-up-high-availability.html

- Follow the same procedure than for the workers to add the nodes back to the Swarm cluster.

- Check Allow administrators to deploy containers on UCP managers or nodes running DTR

- Clear old DTR state

On each dtr node on Marcoussis :
```shell
docker container rm -f $(docker ps -aq --filter "name=dtr-")
docker volume rm $(docker volume ls -q --filter "name=dtr-")
```

- Join new replicas to cluster

Execute these commands on the DTR replica on Tigery : 
```shell
DTR_DOCKERHUB=$(docker ps --filter "name=dtr-registry" --format "{{.Image}}" | cut -d"/" -f1)
DTR_VERSION=$(docker ps --filter "name=dtr-registry" --format "{{.Image}}" | cut -d":" -f2)
DTR_IMAGE=${DTR_DOCKERHUB}/dtr:${DTR_VERSION}

UCP_URL=ucp.pac-amberjack.dockerps.io
USER=totoadmin

# Replace <dtrX> with tne name of the node you want to add
docker run -it --rm ${DTR_IMAGE} join --ucp-node <dtr1> --ucp-insecure-tls --ucp-url $UCP_URL --ucp-username $USER
# Wait for the replica to join the cluster, then join the last replica
docker run -it --rm ${DTR_IMAGE} join --ucp-node <dtr2> --ucp-insecure-tls --ucp-url $UCP_URL --ucp-username $USER
```


[Back to the chronogram](#chronogram)

# Add managers
https://docs.mirantis.com/docker-enterprise/v3.0/dockeree-products/ucp/admin/configure/join-nodes/set-up-high-availability.html

TODO : check labels

:warning:  It is recommended to wipe the old managers on Marcoussis and to create 2 new nodes.  
If you want to use the previous nodes, make sure to wipe datas on them before starting the Docker daemon, as it can create a split brain issue with Etcd, and you will have to start again the DR process from the start.  

- Before adding back the manager to the cluster, you need to wipe the datas in order to prevent it to reconnect automatically to the old etcd/rethinkdb clusters

Execute the following commands on a manager on Marcoussis

```shell
rm -rf /var/lib/docker/*
systemctl start docker
```

- Join the first node as a manager

Execute the following command on the manager on Tigery in order to generate a join token
```shell
docker swarm join-token manager
```

Paste the command on the node you want to join
```shell
docker swarm join <TOKEN HOST:IP>
```

Wait until the manager has fully joind the cluster and is healthy, then repeat operation for the second manager

[Back to the chronogram](#chronogram)

# Check cluster health

Follow the same procedure as before in the chapter [Check current health of the cluster
](#check-current-health-of-the-cluster)

[Back to the chronogram](#chronogram)

# CHECKPOINT DR complete



TODO : Tester un force init swarm et ensuite une restore UCP

TODO : Desactiver admin deploy containers

# Restore Backup

```shell

# manager3
docker container run --rm -it --security-opt label=disable -v /var/run/docker.sock:/var/run/docker.sock --name ucp $UCP_IMAGE uninstall-ucp --interactive
docker container run --rm -i --security-opt label=disable -v /var/run/docker.sock:/var/run/docker.sock --name ucp $UCP_IMAGE restore < /tmp/mybackup.tar
docker node rm <manager1> <manager2>

# dtr3
docker run -it --rm mirantis/dtr:2.7.8 destroy --ucp-insecure-tls

DTR_IMAGE=mirantis/dtr:2.7.8

UCP_URL=ucp.pac-amberjack.dockerps.io
USER=totoadmin

docker run -it --rm ${DTR_IMAGE} restore --ucp-node $(hostname) --ucp-insecure-tls --ucp-url $UCP_URL --ucp-username $USER < dtr-metadata-backup-20201030-15_59_32.tar



```
