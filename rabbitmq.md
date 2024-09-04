# Docs for setting up the rabbitmq cluster with ubuntu server and enabling TLS over protocols such as amqp, and https.


# 1. Changing the hostname of the servers to refer then as the master and worker nodes 

**apt update** :- update packages of your ubuntu server. 

**vi  /etc/hosts** ------------- here add the private ip of the other nodes so that it will help us to differenciate between the worker and the master nodes.

**for Example** 

```shell
  <Private Ip> node-1
  <Private Ip> node-2
  <Private Ip> node-3
```
Then save the hosts file carry out the following commands to change the hostname to node-1 of the master node and node-2 and node-3 of the respective workers.

**sudo hostnamectl set-hostname node-2 –static** --> to change the hostname of the master and worker node as required.

**apt update**:- update the packages so that the hosts file can be updated with the changes made above.

**sudo reboot** ---> Then reboot the instances to make the changes to work as needed.

---

# 2. Shell script for installing the Rabbit MQ and erlang into the servers.

```shell
#!/bin/sh

sudo apt-get install curl gnupg apt-transport-https -y

## Team RabbitMQ's main signing key
curl -1sLf "https://keys.openpgp.org/vks/v1/by-fingerprint/0A9AF2115F4687BD29803A206B73A36E6026DFCA" | sudo gpg --dearmor | sudo tee /usr/share/keyrings/com.rabbitmq.team.gpg > /dev/null
## Community mirror of Cloudsmith: modern Erlang repository
curl -1sLf https://github.com/rabbitmq/signing-keys/releases/download/3.0/cloudsmith.rabbitmq-erlang.E495BB49CC4BBE5B.key | sudo gpg --dearmor | sudo tee /usr/share/keyrings/rabbitmq.E495BB49CC4BBE5B.gpg > /dev/null
## Community mirror of Cloudsmith: RabbitMQ repository
curl -1sLf https://github.com/rabbitmq/signing-keys/releases/download/3.0/cloudsmith.rabbitmq-server.9F4587F226208342.key | sudo gpg --dearmor | sudo tee /usr/share/keyrings/rabbitmq.9F4587F226208342.gpg > /dev/null

## Add apt repositories maintained by Team RabbitMQ
sudo tee /etc/apt/sources.list.d/rabbitmq.list <<EOF
## Provides modern Erlang/OTP releases
##
deb [signed-by=/usr/share/keyrings/rabbitmq.E495BB49CC4BBE5B.gpg] https://ppa1.novemberain.com/rabbitmq/rabbitmq-erlang/deb/ubuntu jammy main
deb-src [signed-by=/usr/share/keyrings/rabbitmq.E495BB49CC4BBE5B.gpg] https://ppa1.novemberain.com/rabbitmq/rabbitmq-erlang/deb/ubuntu jammy main

# another mirror for redundancy
deb [signed-by=/usr/share/keyrings/rabbitmq.E495BB49CC4BBE5B.gpg] https://ppa2.novemberain.com/rabbitmq/rabbitmq-erlang/deb/ubuntu jammy main
deb-src [signed-by=/usr/share/keyrings/rabbitmq.E495BB49CC4BBE5B.gpg] https://ppa2.novemberain.com/rabbitmq/rabbitmq-erlang/deb/ubuntu jammy main

## Provides RabbitMQ
##
deb [signed-by=/usr/share/keyrings/rabbitmq.9F4587F226208342.gpg] https://ppa1.novemberain.com/rabbitmq/rabbitmq-server/deb/ubuntu jammy main
deb-src [signed-by=/usr/share/keyrings/rabbitmq.9F4587F226208342.gpg] https://ppa1.novemberain.com/rabbitmq/rabbitmq-server/deb/ubuntu jammy main

# another mirror for redundancy
deb [signed-by=/usr/share/keyrings/rabbitmq.9F4587F226208342.gpg] https://ppa2.novemberain.com/rabbitmq/rabbitmq-server/deb/ubuntu jammy main
deb-src [signed-by=/usr/share/keyrings/rabbitmq.9F4587F226208342.gpg] https://ppa2.novemberain.com/rabbitmq/rabbitmq-server/deb/ubuntu jammy main
EOF

## Update package indices
sudo apt-get update -y

## Install Erlang packages
sudo apt-get install -y erlang-base \
                        erlang-asn1 erlang-crypto erlang-eldap erlang-ftp erlang-inets \
                        erlang-mnesia erlang-os-mon erlang-parsetools erlang-public-key \
                        erlang-runtime-tools erlang-snmp erlang-ssl \
                        erlang-syntax-tools erlang-tftp erlang-tools erlang-xmerl

## Install rabbitmq-server and its dependencies
sudo apt-get install rabbitmq-server -y --fix-missing
```

@@  To check the version of the rabbitmq server set onto the ubuntu servers run the following command:-

**rabbitmqctl version**

---

# 3. Making the node-1 as the master node and then joining the other nodes to the same to cluster.

**RUN THE FOLLOWING COMMANDS**

@@ on node -1

```vi /var/lib/rabbitmq/.erlang.cookie``` to get the cookie string which rabbitmq will use inorder to communicate with other nodes.
Then restart the rabbitmq service:- systemctl restart rabbitmq-server

@@@ then replace the cookie string which you got above in the same path of which is mentioned above in node2 and node3 

Before that ensure that app has started in the node-1 or else you can start it with the command:- sudo rabbitmqctl start_app

!!! **commands to executed on all the worker nodes that will join the cluster.**

```shell
    sudo rabbitmqctl stop_app
    sudo rabbitmqctl join_cluster rabbit@node-1
    sudo rabbitmqctl start_app
```

> **_NOTE:_** :- If the worker nodes do not join with the above command then in that case check that following ports are opened on firewall settings of your instances or servers. 
1. **4369**: epmd, a helper discovery daemon used by RabbitMQ nodes and CLI tools
2. **6000-6500**: used by RabbitMQ Stream replication
3. **25672**: used for inter-node and CLI tools communication (Erlang distribution server port) and is allocated from a dynamic range (limited to a single port by default, computed as AMQP port + 20000). Unless external connections on these ports are really necessary (e.g. the cluster uses federation or CLI tools are used on machines outside the subnet), these ports should not be publicly exposed. See networking guide for details.
4. **35672-35682**: used by CLI tools (Erlang distribution client ports) for communication with nodes and is allocated from a dynamic range (computed as server distribution port + 10000 through server distribution port + 10010).
5. **5671** :- amqp/ssl port
6. **15671** :- https /ssl port
7. **5672** :- amqp port
8. **15672** :- http port

> **_NOTE:_** :- You can also check other ports according to the need of your application.

**Run the following command to check the cluster status and verify the nodes.**

```shell
   rabbitmqctl cluster_status
```

# 4. Creating the a user with admin permission and deleting the default guest user.

```shell
    # Run the following commands on master node:-  
    sudo rabbitmqctl add_user <username> <password>
    sudo rabbitmqctl set_user_tags <username> administrator
    # First ".*" for configure permission on every entity
    # Second ".*" for write permission on every entity
    # Third ".*" for read permission on every entity
    rabbitmqctl set_permissions -p "custom-vhost" "username" ".*" ".*" ".*"
    rabbitmqctl delete_user guest
    sudo rabbitmqctl set_policy ha-all "." '{"ha-mode":"all"}'
    rabbitmqctl list_policies
    sudo rabbitmq-plugins enable rabbitmq_management # to be applied on all nodes as the this command to help us to see the statistics of all node present in the same cluster.
    rabbitmqctl -q --formatter pretty_table list_feature_flags # to check all the flags that are enabled
```

@@ After applying all the command you will able to access the rabbitmq cluster ip:15672

# 5. Inoder to setup amqp/ssl and https/ssl on the rabbitmq cluster.

**Run the following commands**

```shell
    cd /etc/rabbitmq
    mkdir ssl 
    cd ssl
    openssl genrsa -out ca_key.pem 2048
    openssl req -new -x509 -key ca_key.pem -out ca_certificate.pem -days 3650
    openssl genrsa -out server_key.pem 2048
    openssl req -new -key server_key.pem -out server_req.pem
    openssl x509 -req -in server_req.pem -CA ca_certificate.pem -CAkey ca_key.pem -CAcreateserial -out server_certificate.pem -days 3650
```

@@ make a config file named rabbitmq.conf 

```shell

# RabbitMQ configuration file at /etc/rabbitmq/rabbitmq.conf

listeners.ssl.default = 5671
ssl_options.cacertfile = /path/to/ca_cert.pem
ssl_options.certfile = /path/to/server_cert.pem
ssl_options.keyfile = /path/to/server_key.pem
ssl_options.verify = verify_peer
ssl_options.fail_if_no_peer_cert = false

management.listener.port = 15671
management.listener.ssl = true
management.listener.ssl_opts.cacertfile = /path/to/ca_cert.pem
management.listener.ssl_opts.certfile = /path/to/server_cert.pem
management.listener.ssl_opts.keyfile = /path/to/server_key.pem

```

> **_NOTE:_** :- You can also check the rabbitmq dashboard on https://<IP_address_your_server>:15671




 
