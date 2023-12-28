
## TDEngine Cluster - Docker Swarm - Docker service

## Table of Contents
* [Introduction](#Introduction)
* [Checking_Server_Setup](#Checking_Server_Setup)
* [TDEngine](#TDEngine)
    * [Storage](#Storage)
    * [docker_leader](#docker_leader)
    * [overlay_network](#overlay_network)
    * [Create_tdengine_services](#Create_tdengine_services)
    * [accessing_TAOS](#accessing_TAOS)
    * [Create_taosAdapters_service](#Create_taosAdapters_service)



## Introduction
This Guide is for setting up a <b>TDEngine Cluster</b> using docker services on a docker swarm server cluster.
<br>
This guide assumes you have a running Docker Swarm on 3 or more unique/individual linux servers running docker in swarm mode with workers on each of the servers we will be installing TDengine doceker container on.
<br>

## Checking_Server_Setup
Bring up a terminal on the Docker Swarm Manager Leader
<br>
In terminal:
```
$ docker node ls
```

You should see your Swarm nodes. Again you need 3 or more nodes running with worker nodes on each. Currently I have 3 manager nodes with workers on each.
```
foo@foo1:~$ docker node ls
ID                            HOSTNAME       STATUS    AVAILABILITY   MANAGER STATUS   ENGINE VERSION
jakycx7c6osl85zihr72msw72 *   foo1.bar.com   Ready     Active         Leader           24.0.7
f1uozkzjfsct5bu2xr3620ih2     foo2.bar.com   Ready     Active         Reachable        24.0.7
sf7zwprpt0r6rdpy8pl0a0ste     foo3.bar.com   Ready     Active         Reachable        24.0.7
```

You can also tell by the <b>*</b> after the ID that im on the Leader machiene. If the <b>*</b> is on another machiene make sure you STOP and loginto that one before continuing. Also STOP and correct/fix you Docker swarm setup if you do not have 3 Individual nodes with workers avalible on each one.
<br>

## TDEngine

##### Storage
First lets setup persistent storage for each unique TDEngine instance we will be running. You will need todo this for each individual server! Don't run these commands all on the same server!
<br>
For this example I have 3 servers. foo1.bar.com, foo2.bar.com, and foo3.bar.com
<br>

In terminal: On foo1.bar.com (your server name for the server you will install TDEngine1)
```
foo@foo1:~$ mkdir -p $HOME/docker/cluster/TDEngine/tdengine1/var/lib/taos
```
<br>
In terminal: On foo2.bar.com (your server name for the server you will install TDEngine2)
```
foo@foo2:~$ mkdir -p $HOME/docker/cluster/TDEngine/tdengine2/var/lib/taos
```
<br>
In terminal: On foo3.bar.com (your server name for the server you will install TDEngine3)
```
foo@foo3:~$ mkdir -p $HOME/docker/cluster/TDEngine/tdengine3/var/lib/taos
```
<br>

##### docker_leader
Now connect to the terminal that is running your docker manager leader as outlined above in Checking_Server_Setup.
We will only use the terminal on the leader for the rest of this documentation.
<br>

##### overlay_network
I like to run my TDEngine database protected inside a docker only network. I will show you how to expose it later with a taosadapter service we will create.
Also any other docker services that I wan't talking to TDEngine or the taosadapter make sure you have them join this overlay network when you create them.
<br>
Create an overlay network on the docker manager node leader
```
foo@foo1:~$ docker network create --driver overlay internalnetwork
```

##### Create_tdengine_services
Create 3 individual tdengine services. (done on swarm manager leader's terminal)
<br>
Don't worry about the options, IE --replicas 1 \, too much. If you want to know what every thing does in detail see ---> [https://docs.docker.com/engine/reference/commandline/service_create/](https://docs.docker.com/engine/reference/commandline/service_create/)
<br>
Make sure to change the version to the one you want to use for all services. IE tdengine:3.2.2.0
<br>
Start service on server foo1.bar.com with a --constraint node.hostname==foo1.bar.com so it runs on that server
```
foo@foo1:~$ docker service create \
--replicas 1 \
--replicas-max-per-node 1 \
--log-opt max-size=5m \
--log-opt max-file=3 \
--network internalnetwork \
--constraint node.hostname==foo1.bar.com \
--name "tdengine1" \
--mount type=bind,source="$HOME/docker/cluster/TDEngine/tdengine1/var/lib/taos",destination=/var/lib/taos \
--env TAOS_FQDN=tdengine1 \
--env TAOS_FIRST_EP=tdengine1 \
--env TAOS_SECOND_EP=tdengine2 \
--env TZ=UTC \
--env TAOS_DISABLE_ADAPTER=true \
tdengine/tdengine:3.2.2.0
```
<br>

Start service on server foo1.bar.com with a --constraint node.hostname==foo2.bar.com so it runs on that server
```
foo@foo1:~$ docker service create \
--replicas 1 \
--replicas-max-per-node 1 \
--log-opt max-size=5m \
--log-opt max-file=3 \
--network internalnetwork \
--constraint node.hostname==foo2.bar.com \
--name "tdengine2" \
--mount type=bind,source="$HOME/docker/cluster/TDEngine/tdengine2/var/lib/taos",destination=/var/lib/taos \
--env TAOS_FQDN=tdengine2 \
--env TAOS_FIRST_EP=tdengine1 \
--env TAOS_SECOND_EP=tdengine2 \
--env TZ=UTC \
--env TAOS_DISABLE_ADAPTER=true \
tdengine/tdengine:3.2.2.0
```
<br>

Start service on server foo1.bar.com with a --constraint node.hostname==foo3.bar.com so it runs on that server
```
foo@foo1:~$ docker service create \
--replicas 1 \
--replicas-max-per-node 1 \
--log-opt max-size=5m \
--log-opt max-file=3 \
--network internalnetwork \
--constraint node.hostname==foo3.bar.com \
--name "tdengine3" \
--mount type=bind,source="$HOME/docker/cluster/TDEngine/tdengine1/var/lib/taos",destination=/var/lib/taos \
--env TAOS_FQDN=tdengine3 \
--env TAOS_FIRST_EP=tdengine1 \
--env TAOS_SECOND_EP=tdengine2 \
--env TZ=UTC \
--env TAOS_DISABLE_ADAPTER=true \
tdengine/tdengine:3.2.2.0
```
<br>

Issues/Check that its running properly you can use these commands
```
foo@foo1:~$ docker service ls
foo@foo1:~$ docker service logs -f tdengine1
foo@foo1:~$ docker service logs -f tdengine2
foo@foo1:~$ docker service logs -f tdengine3
```
<br>

##### accessing_TAOS
We need to check that our DNODE's are avalible and running. Also Lets create 2 more MNODES as we can have 3 managers in a TDEngine Swarm'
<br>
On swarm manager leader's terminal
```
foo@foo1:~$ docker ps
```
<br>
You should see the container ID for tdengine1
```
CONTAINER ID   IMAGE                       COMMAND                  CREATED        STATUS        PORTS      NAMES
4751e25af602   tdengine/tdengine:3.2.2.0   "/tini -- /usr/bin/e…"   20 hours ago   Up 20 hours              tdengine1.1.tvc4mdliedgk8ytar0fk3hx2l
```
<br>
Make sure to change the container ID to the one you got in the output above
```
foo@foo1:~$ docker exec -it 4751e25af602 taos
```
<br>
You should now be in TAOS command line. Should look like this.
```
Welcome to the TDengine Command Line Interface, Client Version:3.2.2.0
Copyright (c) 2023 by TDengine, all rights reserved.

  ********************************  Tab Completion  ************************************
  *   The TDengine CLI supports tab completion for a variety of items,                 *
  *   including database names, table names, function names and keywords.              *
  *   The full list of shortcut keys is as follows:                                    *
  *    [ TAB ]        ......  complete the current word                                *
  *                   ......  if used on a blank line, display all supported commands  *
  *    [ Ctrl + A ]   ......  move cursor to the st[A]rt of the line                   *
  *    [ Ctrl + E ]   ......  move cursor to the [E]nd of the line                     *
  *    [ Ctrl + W ]   ......  move cursor to the middle of the line                    *
  *    [ Ctrl + L ]   ......  clear the entire screen                                  *
  *    [ Ctrl + K ]   ......  clear the screen after the cursor                        *
  *    [ Ctrl + U ]   ......  clear the screen before the cursor                       *
  **************************************************************************************

Server is Community Edition.

taos> 
```
<br>
Lets check that the dnodes are up and running
```
taos> show dnodes;
```
<br>
You should see all of your DNODE's and the 3 servers. If not STOP!!!! do not continue. you have missed somting in this guide. Start at the begining and read carefully to find what step you missed.
<br>
it should look like this.
```
     id      |            endpoint            | vnodes | support_vnodes |    status    |       create_time       |       reboot_time       |              note              |
=============================================================================================================================================================================
           1 | tdengine1:6030                 |      3 |              8 | ready        | 2023-12-27 21:06:55.471 | 2023-12-27 21:06:55.287 |                                |
           2 | tdengine2:6030                 |      3 |              8 | ready        | 2023-12-27 21:07:22.529 | 2023-12-27 21:07:22.921 |                                |
           3 | tdengine3:6030                 |      3 |              8 | ready        | 2023-12-27 21:07:46.974 | 2023-12-27 21:07:47.377 |                                |
Query OK, 3 row(s) in set (0.122209s)

```
<br>
Adding MNODE's. We should only see the automaticaly created MNODE on DNODE 1 to start
```
taos> SHOW MNODES;
```
<br>
Lets create the other 2 MNODE's
```
taos> CREATE MNODE ON DNODE 2;
taos> CREATE MNODE ON DNODE 3;
```
<br>
Check to see if managers were created
```
taos> SHOW MNODES;
```
<br>
We should see this
```
     id      |            endpoint            |      role      |   status    |       create_time       |        role_time        |
==================================================================================================================================
           1 | tdengine1:6030                 | follower       | ready       | 2023-12-27 21:06:55.491 | 2023-12-28 09:15:51.849 |
           2 | tdengine2:6030                 | follower       | ready       | 2023-12-27 21:09:04.384 | 2023-12-28 09:15:51.859 |
           3 | tdengine3:6030                 | leader         | ready       | 2023-12-27 21:09:13.180 | 2023-12-28 09:15:51.894 |
Query OK, 3 row(s) in set (0.120404s)

```
<br>
Now if a manager goes down another will be promoted to the leader and your TDEngine will continue to work while you have time to get that server back up
<br>
Exit TAOS. This will bring you back to your servers CLI
```
taos> exit
```
<br>

##### Create_taosAdapters_service
We want access to our tdengine cluster through the taosAdapter able to scale and be fault talerant. Lets create a service with 3 replications 1 on each docker node server.
```
docker service create \
--replicas 3 \
--replicas-max-per-node 1 \
--update-delay 10s \
--update-failure-action pause \
--update-parallelism 1 \
--rollback-delay 10s \
--rollback-failure-action pause \
--rollback-parallelism 1 \
--log-opt max-size=5m \
--log-opt max-file=3 \
--network internalnetwork \
--publish published=6041,target=6041,protocol=tcp
--publish published=6041,target=6041,protocol=udp
--name "taosadapter" \
--entrypoint taosadapter \
--env TAOS_FIRST_EP=tdengine0 \
--env TAOS_SECOND_EP=tdengine1 \
--env TZ=UTC \
tdengine/tdengine:3.2.2.0
```
<br>

If you don't want/need external access because all your docker services are on the same docker network just remove the --publish lines.
```
docker service create \
--replicas 3 \
--replicas-max-per-node 1 \
--update-delay 10s \
--update-failure-action pause \
--update-parallelism 1 \
--rollback-delay 10s \
--rollback-failure-action pause \
--rollback-parallelism 1 \
--log-opt max-size=5m \
--log-opt max-file=3 \
--network internalnetwork \
--name "taosadapter" \
--entrypoint taosadapter \
--env TAOS_FIRST_EP=tdengine0 \
--env TAOS_SECOND_EP=tdengine1 \
--env TZ=UTC \
tdengine/tdengine:3.2.2.0
```
<br>
Check if service is ok and running
```
foo@foo1:~$ docker service ls
foo@foo1:~$ docker service logs -f taosadapter
```

Testing taosadapter works using curl
<br>
The TAOS docker service we created called "taosadapter" handles all of the trafic routing to the 3 replicas we created. We don't try to connect to the replicas we connect to the service and it handles the routing.
<br>
From inside the overlay network ( the internal docker network ) we created. You can access the taosadapeter from another service using the name of the container.
```
url = taosadapter:6041/rest/sql/yourdatabasename?tz=America/Detroit
```
<br>
From outside the overlay network ( if you have enabled portfowarding ) You can access the taosadapeter using the localhost
```
url = localhost:6041/rest/sql/yourdatabasename?tz=America/Detroit
or
url = 127.0.0.1:6041/rest/sql/yourdatabasename?tz=America/Detroit
```

To understand how to use the api connecting to the TAOS adapter see ---> [https://docs.tdengine.com/reference/rest-api/](https://docs.tdengine.com/reference/rest-api/)


END of Document..................










