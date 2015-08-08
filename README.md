# nodejs_apps_to_docker_balanced
To deploy docker node app on Nginx Load balancer
This repository is a fork for https://github.com/bellycard/docker-loadbalancer

We add this extra configuration:
```yaml
privileged: true
```

We start the leader consul on the docker0 interface:
```yaml
docker0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.42.1  netmask 255.255.0.0  broadcast 0.0.0.0
```

After the execution of:
```yaml
[myusertest@vm14 nodejs_apps_to_docker_balanced]$ sudo docker-compose up

With this configuration all the services started without problems:
172.17.0.7  /nodejsappstodockerbalanced_registrator_1 gliderlabs/registrator true fa363950c7bd8a51f0584aaf8fb8cb0b80ea59405cf5fd5b16b47cae5618a13e
172.17.0.6  /nodejsappstodockerbalanced_lb_1 nodejsappstodockerbalanced_lb true a9a666b5b78304f029effb1cd19f21e40e8b2071698c03259e5b8e42fb101a3e
172.17.0.2  /nodejsappstodockerbalanced_consul_1 progrium/consul:latest true 4268809eaa8295a80f926f8321efebfeb089010ded7c514a35e58ebcda148f9b
172.17.0.1  /nodejsappstodockerbalanced_app_1 dsusanibar/nodejscloudport:latest true 19fd10ff602c0936befc5aa8c524612ea5ffa37b6606fe15091211e276159546

[myusertest@vm14 nodejs_apps_to_docker_balanced]$ sudo docker ps -a

CONTAINER ID        IMAGE                                  COMMAND                CREATED             STATUS              PORTS                                                                                                                                NAMES
fa363950c7bd        gliderlabs/registrator:latest          "/bin/registrator co   About an hour ago   Up About an hour                                                                                                                                         nodejsappstodockerbalanced_registrator_1
a9a666b5b783        nodejsappstodockerbalanced_lb:latest   "/usr/bin/runsvdir /   About an hour ago   Up About an hour    0.0.0.0:80->80/tcp, 443/tcp                                                                                                          nodejsappstodockerbalanced_lb_1      
4268809eaa82        progrium/consul:latest                 "/bin/start -server    About an hour ago   Up About an hour    53/tcp, 0.0.0.0:8300->8300/tcp, 0.0.0.0:8400->8400/tcp, 8301-8302/tcp, 0.0.0.0:8500->8500/tcp, 8301-8302/udp, 0.0.0.0:8600->53/udp   nodejsappstodockerbalanced_consul_1  
19fd10ff602c        dsusanibar/nodejscloudport:latest      "node /src/index.js    About an hour ago   Up About an hour    0.0.0.0:32768->8080/tcp                                                                                                              nodejsappstodockerbalanced_app_1     
```

Test:
- If we call directly my Node JS services that is running on my container, it was resolved without problems:
```yaml
[myusertest@vm14 nodejs_apps_to_docker_balanced]$ curl 172.17.0.1:8080/bluegreen
Hello world to test blue green deployment!! Server: ::, Port: 8080

[myusertest@vm14 nodejs_apps_to_docker_balanced]$ curl localhost:32768/bluegreen
Hello world to test blue green deployment!! Server: ::, Port: 8080
```
, but if we call to my Nginx Proxy pass, it is not resolved, but the request is arrived to the container as we could see:
```yaml
[myusertest@vm14 nodejs_apps_to_docker_balanced]$ curl localhost:80/bluegreen
<html>
<head><title>502 Bad Gateway</title></head>
<body bgcolor="white">
<center><h1>502 Bad Gateway</h1></center>
<hr><center>nginx/1.7.12</center>
</body>
</html>
```
The request is arrived to the Nginx container but is not completed, we saw on the docker-compose logs:
```yaml
lb_1          | 2015/08/08 23:23:16 [error] 21#0: *19 connect() failed (113: No route to host) while connecting to upstream, client: 172.17.42.1, server: , request: "GET /bluegreen HTTP/1.1", upstream: "http://192.168.1.14:32768/bluegreen", host: "localhost"
lb_1          | 172.17.42.1 - - [08/Aug/2015:23:23:16 +0000] "GET /bluegreen HTTP/1.1" 502 173 "-" "curl/7.29.0" "-"
```
To solve this problem, there are many options, for example:
- http://blog.gnu-designs.com/howto-enable-docker-api-through-firewalld-on-centos-7-x-el7/
- http://blog.simulakrum.org/?p=243
- http://unix.stackexchange.com/questions/199966/how-to-configure-centos-7-firewalld-to-allow-docker-containers-free-access-to-th

At the end, our project started correctly after the execution of this commands:
```yaml
[myusertest@vm14 nodejs_apps_to_docker_balanced]$ sudo firewall-cmd --permanent --zone=trusted --add-interface=docker0
success
[myusertest@vm14 nodejs_apps_to_docker_balanced]$ sudo firewall-cmd --permanent --zone=trusted --add-port=4243/tcp
success
[myusertest@vm14 nodejs_apps_to_docker_balanced]$ sudo firewall-cmd --reload
success
[myusertest@vm14 nodejs_apps_to_docker_balanced]$ curl localhost:80/bluegreen
Hello world to test blue green deployment!! Server: ::, Port: 8080
```

With this new configuration all the services started without problems, log example:
```yaml
[myusertest@vm14 nodejs_apps_to_docker_balanced]$ sudo docker-compose up

Creating nodejsappstodockerbalanced_lb_1...
Creating nodejsappstodockerbalanced_registrator_1...
Attaching to nodejsappstodockerbalanced_app_1, nodejsappstodockerbalanced_consul_1, nodejsappstodockerbalanced_lb_1, nodejsappstodockerbalanced_registrator_1
consul_1      | ==> WARNING: Bootstrap mode enabled! Do not enable unless necessary
app_1         | Example app listening at http://:::8080
consul_1      | ==> WARNING: It is highly recommended to set GOMAXPROCS higher than 1
consul_1      | ==> Starting raft data migration...
consul_1      | ==> Starting Consul agent...
consul_1      | ==> Starting Consul agent RPC...
consul_1      | ==> Consul agent running!
consul_1      |          Node name: '4268809eaa82'
consul_1      |         Datacenter: 'dc1'
consul_1      |             Server: true (bootstrap: true)
consul_1      |        Client Addr: 0.0.0.0 (HTTP: 8500, HTTPS: -1, DNS: 53, RPC: 8400)
consul_1      |       Cluster Addr: 192.168.1.14 (LAN: 8301, WAN: 8302)
consul_1      |     Gossip encrypt: false, RPC-TLS: false, TLS-Incoming: false
consul_1      |              Atlas: <disabled>
consul_1      |
consul_1      | ==> Log data will now stream in as it occurs:
consul_1      |
consul_1      |     2015/08/08 21:44:13 [INFO] serf: EventMemberJoin: 4268809eaa82 192.168.1.14
consul_1      |     2015/08/08 21:44:13 [INFO] serf: EventMemberJoin: 4268809eaa82.dc1 192.168.1.14
consul_1      |     2015/08/08 21:44:13 [INFO] raft: Node at 192.168.1.14:8300 [Follower] entering Follower state
consul_1      |     2015/08/08 21:44:13 [INFO] consul: adding server 4268809eaa82 (Addr: 192.168.1.14:8300) (DC: dc1)
consul_1      |     2015/08/08 21:44:13 [INFO] consul: adding server 4268809eaa82.dc1 (Addr: 192.168.1.14:8300) (DC: dc1)
consul_1      |     2015/08/08 21:44:13 [ERR] agent: failed to sync remote state: No cluster leader
consul_1      |     2015/08/08 21:44:14 [WARN] raft: Heartbeat timeout reached, starting election
consul_1      |     2015/08/08 21:44:14 [INFO] raft: Node at 192.168.1.14:8300 [Candidate] entering Candidate state
consul_1      |     2015/08/08 21:44:15 [INFO] raft: Election won. Tally: 1
consul_1      |     2015/08/08 21:44:15 [INFO] raft: Node at 192.168.1.14:8300 [Leader] entering Leader state
consul_1      |     2015/08/08 21:44:15 [INFO] consul: cluster leadership acquired
consul_1      |     2015/08/08 21:44:15 [INFO] consul: New leader elected: 4268809eaa82
consul_1      |     2015/08/08 21:44:15 [INFO] raft: Disabling EnableSingleNode (bootstrap)
consul_1      |     2015/08/08 21:44:15 [INFO] consul: member '4268809eaa82' joined, marking health alive
consul_1      |     2015/08/08 21:44:17 [INFO] agent: Synced service 'consul'
lb_1          | nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
lb_1          | nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
lb_1          | nginx: configuration file /etc/nginx/nginx.conf test is successful
lb_1          | nginx: configuration file /etc/nginx/nginx.conf test is successful
registrator_1 | 2015/08/08 21:46:56 registrator: Using consul registry backend at consul://consul:8500
registrator_1 | 2015/08/08 21:46:56 registrator: ignored: fa363950c7bd no published ports
registrator_1 | 2015/08/08 21:46:56 registrator: ignored a9a666b5b783 port 443 not published on host
consul_1      |     2015/08/08 21:46:56 [WARN] Service name "nodejsappstodockerbalanced_lb-80" will not be discoverable via DNS due to invalid characters. Valid characters include all alpha-numerics and dashes.
consul_1      |     2015/08/08 21:46:56 [INFO] agent: Synced service 'fa363950c7bd:nodejsappstodockerbalanced_lb_1:80'
registrator_1 | 2015/08/08 21:46:56 registrator: added: a9a666b5b783 fa363950c7bd:nodejsappstodockerbalanced_lb_1:80
registrator_1 | 2015/08/08 21:46:56 registrator: ignored 4268809eaa82 port 53 not published on host
consul_1      |     2015/08/08 21:46:56 [INFO] agent: Synced service 'fa363950c7bd:nodejsappstodockerbalanced_consul_1:8300'
registrator_1 | 2015/08/08 21:46:56 registrator: added: 4268809eaa82 fa363950c7bd:nodejsappstodockerbalanced_consul_1:8300
registrator_1 | 2015/08/08 21:46:56 registrator: ignored 4268809eaa82 port 8301 not published on host
registrator_1 | 2015/08/08 21:46:56 registrator: ignored 4268809eaa82 port 8301 not published on host
registrator_1 | 2015/08/08 21:46:56 registrator: ignored 4268809eaa82 port 8302 not published on host
consul_1      |     2015/08/08 21:46:56 [INFO] agent: Synced service 'fa363950c7bd:nodejsappstodockerbalanced_consul_1:8400'
registrator_1 | 2015/08/08 21:46:56 registrator: added: 4268809eaa82 fa363950c7bd:nodejsappstodockerbalanced_consul_1:8400
consul_1      |     2015/08/08 21:46:56 [INFO] agent: Synced service 'fa363950c7bd:nodejsappstodockerbalanced_consul_1:53:udp'
registrator_1 | 2015/08/08 21:46:56 registrator: added: 4268809eaa82 fa363950c7bd:nodejsappstodockerbalanced_consul_1:53:udp
registrator_1 | 2015/08/08 21:46:56 registrator: ignored 4268809eaa82 port 8302 not published on host
consul_1      |     2015/08/08 21:46:56 [INFO] agent: Synced service 'fa363950c7bd:nodejsappstodockerbalanced_consul_1:8500'
registrator_1 | 2015/08/08 21:46:56 registrator: added: 4268809eaa82 fa363950c7bd:nodejsappstodockerbalanced_consul_1:8500
consul_1      |     2015/08/08 21:46:56 [INFO] agent: Synced service 'fa363950c7bd:nodejsappstodockerbalanced_app_1:8080'
registrator_1 | 2015/08/08 21:46:56 registrator: added: 19fd10ff602c fa363950c7bd:nodejsappstodockerbalanced_app_1:8080
registrator_1 | 2015/08/08 21:46:56 registrator: Listening for Docker events...
```

It was tested on Centox 7
Linux pchost 3.10.0-123.13.2.el7.x86_64 x86_64 GNU/Linux


