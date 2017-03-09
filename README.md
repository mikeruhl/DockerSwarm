# DockerSwarm
Configuration to stand up a highly available docker swarm with Consul as the discovery service and Registrator as the service registry bridge.

---

## **!!!Warning!!!**
This script so far does not implement security.  It is open and therefore not intended for production.  Later, I will add certificate support to these.  This configuration is good for sandboxing in something like VirtualBox.

---

For my test setup, I have 6 servers: 3 managers and 3 nodes. The server names correspond to their IPs starting with server1 = 192.168.56.51.  I've set it up so server1-server3 are my managers and server4-6 are my nodes. In the real world, I would have at least one manager in each failure domain and then n nodes depending upon demand.  This setup starts after installing Ubuntu on each machine and installing docker.  All I did for this was setup 1 machine and clone it 5 times, changing my static IP settings in /etc/network/interfaces and changing my names in /etc/hosts and /etc/hostname

# 1. Run the Discovery Service :
## Consul Server
The first thing we'll set up is the discovery service.  In this demo I'm using [Consul](https://www.consul.io/).  You can find documentation on Docker Hub [here](https://hub.docker.com/_/consul/).
For each run command, I name the containers differently.

For this purpose, the main manager always has to be up because I'm joining on that IP for every other server.  This is not production quality and you'll want to put a load balancer out in front of your managers and point the IP to that instead so that no matter what manager goes down, you'll be safe.

- Main manager (1) *Example uses server1*
```
 docker run --restart=unless-stopped -d -h consul1 --name consul1 -v /mnt:/data \
-p 192.168.56.51:8300:8300 \
-p 192.168.56.51:8301:8301 \
-p 192.168.56.51:8301:8301/udp \
-p 192.168.56.51:8302:8302/udp \
-p 192.168.56.51:8302:8302 \
-p 192.168.56.51:8400:8400 \
-p 192.168.56.51:8500:8500 \
-p 172.17.0.1:53:53/udp \
progrium/consul -server -advertise 192.168.56.51 -bootstrap-expect 3
```
- Other managers (2) *Example uses server2*
```
docker run --restart=unless-stopped -d -h consul2 --name consul2 -v /mnt:/data \
-p 192.168.56.52:8300:8300 \
-p 192.168.56.52:8301:8301 \
-p 192.168.56.52:8301:8301/udp \
-p 192.168.56.52:8302:8302/udp \
-p 192.168.56.52:8302:8302 \
-p 192.168.56.52:8400:8400 \
-p 192.168.56.52:8500:8500 \
-p 172.17.0.1:53:53/udp \
progrium/consul -server -advertise 192.168.56.52 -join 192.168.56.51
```
- Nodes (3) *Example uses server6* 
```
docker run --restart=unless-stopped -d -h consul-agt3 --name consul-agt3 \
-p 8300:8300 \
-p 8301:8301 \
-p 8301:8301/udp \
-p 8302:8302/udp \
-p 8302:8302 \
-p 8400:8400 \
-p 8500:8500 \
-p 8600:8600/udp \
progrium/consul -rejoin -advertise 192.168.56.56 -join 192.168.56.51
```
The run commands set up the port mappings and advertise the machine's IP.  Notice the last port mapping is different: that's DNS to AWS's DNS servers.  So that needs to be changed as well to your own DNS server.
For the main manager, the bootstrap-expect command tells it how many managers to expect.  This is independent from the number of nodes you add.

You can check your work with this handy command:
```
curl http://192.168.56.51:8500/v1/catalog/nodes | python -m json.tool
```

Here's my output if you're following along:
```
mike@server1:~$ curl http://192.168.56.51:8500/v1/catalog/nodes | python -m json.tool
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   283  100   283    0     0  32554      0 --:--:-- --:--:-- --:--:-- 56600
[
    {
        "Address": "192.168.56.54",
        "Node": "consul-agt1"
    },
    {
        "Address": "192.168.56.55",
        "Node": "consul-agt2"
    },
    {
        "Address": "192.168.56.56",
        "Node": "consul-agt3"
    },
    {
        "Address": "192.168.56.51",
        "Node": "consul1"
    },
    {
        "Address": "192.168.56.52",
        "Node": "consul2"
    },
    {
        "Address": "192.168.56.53",
        "Node": "consul3"
    }
]
```
All 6 servers up and talking to each other.  Fantastic!

You can also go into a Consul container and look at the status by using Consul commands:
```
docker exec -it consul1 bash
consul members
```
With the output:
```
mike@server1:~$ docker exec -it consul1 bash
bash-4.3# consul members
Node         Address             Status  Type    Build  Protocol  DC
consul1      192.168.56.51:8301  alive   server  0.5.2  2         dc1
consul3      192.168.56.53:8301  alive   server  0.5.2  2         dc1
consul2      192.168.56.52:8301  alive   server  0.5.2  2         dc1
consul-agt1  192.168.56.54:8301  alive   client  0.5.2  2         dc1
consul-agt2  192.168.56.55:8301  alive   client  0.5.2  2         dc1
consul-agt3  192.168.56.56:8301  alive   client  0.5.2  2         dc1

```

## 2. Swarm Server
For the swarm servers, I named each manager mgr1-mgr3.  The advertise IP changes with each machine, however, the consul address will be your main consul manager.
For the nodes, I kept it simple, each node has the same container name of "join"

- Manager (3) *Example users server1*
``` 
docker run --restart=unless-stopped -h mgr1 --name mgr1 -d -p 3375:2375 swarm manage --replication --advertise 192.168.56.51:3375 consul://192.168.56.51:8500/
```
- Node (3) *Example uses server6*
```
docker run -d -h join --name join swarm join --advertise 192.168.56.56:2375 consul://192.168.56.51:8500/
```

To check your work, you can view the logs on each machine.
```
mike@server1:~$ docker logs mgr1
time="2017-03-08T12:39:27Z" level=info msg="Initializing discovery without TLS"
time="2017-03-08T12:39:27Z" level=info msg="Listening for HTTP" addr=":2375" proto=tcp
time="2017-03-08T12:39:27Z" level=info msg="Leader Election: Cluster leadership lost"
time="2017-03-08T12:39:27Z" level=info msg="New leader elected: 192.168.56.52:3375"
```
In my logs, I had brought down the first server set up and restarted it to regenerate some logs.  That's why server2 is the new leader.

You can also view the nodes with the swarm list command:
```

mike@server1:~$ docker run swarm list consul://192.168.56.51:8500/
time="2017-03-09T12:44:23Z" level=info msg="Initializing discovery without TLS"
192.168.56.54:2375
192.168.56.55:2375
192.168.56.56:2375
```
More information on that can be found [here](https://docs.docker.com/swarm/reference/list/#discovery--discovery-backend)
## 3. Registrator
This will register services that come online in Docker.  The final piece!  Same on all 6 servers.
```
docker run --restart=unless-stopped -d --name reg -h reg -v /var/run/docker.sock:/tmp/docker.sock gliderlabs/registrator:latest consul://192.168.56.51:8500
```

## 4. Validation
So let's register a service and make sure everyone knows about it.  Let's use nginx to keep things simple:

First, I'll run nginx, then I'll hit consul's services endpoint to validate it's there.

```
mike@server1:~$ docker run -d --name web1 -p 80:80 nginx
Unable to find image 'nginx:latest' locally
latest: Pulling from library/nginx
693502eb7dfb: Already exists
6decb850d2bc: Pull complete
c3e19f087ed6: Pull complete
Digest: sha256:52a189e49c0c797cfc5cbfe578c68c225d160fb13a42954144b29af3fe4fe335
Status: Downloaded newer image for nginx:latest
5e394f9de6dafd4dd6f91a9943a37362bf316126c6666a190d8719d676067272
mike@server1:~$ curl http://192.168.56.51:8500/v1/catalog/services | python -m json.tool
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   175  100   175    0     0  17793      0 --:--:-- --:--:-- --:--:-- 29166
{
    "consul": [],
    "consul-53": [
        "udp"
    ],
    "consul-8300": [],
    "consul-8301": [
        "udp"
    ],
    "consul-8302": [
        "udp"
    ],
    "consul-8400": [],
    "consul-8500": [],
    "consul-8600": [
        "udp"
    ],
    "nginx-80": [],
    "swarm": []
}
```

And we're good, now to lock this down...