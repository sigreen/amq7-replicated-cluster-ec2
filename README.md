# AMQ 7.x Master / Slave Replicated Cluster on EC2


This project demonstrates how to setup a three-node AMQ 7 Master / Slave cluster (using shared-nothing replication) within a single Amazon EC2 Availability Zone (AZ).  Please note that although this architecture *may* work cross-AZ, it is not recommended nor supported.

![Master Slave Architecture](images/MasterSlaveReplicatedEC2.png)

## Prerequisites

The following product / OS prerequisites exist:

* RHEL 7.3 (with an activated subscription)
* EC2 c4.xlarge instance type
* AMQ 7.3.x+ Broker (install from [here](https://access.redhat.com/jbossnetwork/restricted/softwareDownload.html?softwareId=67801))
* The legacy ActiveMQ client JAR which supports OpenWire failover [here](https://mvnrepository.com/artifact/org.apache.activemq/activemq-all/5.14.5)

## Procedure

### Setup EC2

To get things started, we need to setup three separate EC2 instances.

1. Provision three RHEL 7.3, c4.xlarge instances.
2. Associate three separate Elastic IP's to each instance.  Name them Master, Slave One, and Slave Two.

![EC2 Instances](images/ec2Instance.png)

3. As part of the setup process, be sure to create a Security Group that opens up inbound ports 61616, 22, 80, and 8161.  All outbound traffic should be left open.

![Security Group](images/securityGroup.png)

4. Startup each instance, and secure copy the AMQ 7 binary to each machine:

```
scp -i ~/Downloads/ec2-ssh.pem ~/Downloads/amq-broker-7.3.0-bin.zip ec2-user@XX.XX.XX.XX.compute-1.amazonaws.com:
```

5. For each instance, unzip the binary and run the following command from the `amq-broker-7.3.0-bin` directory:

```
./bin/artemis create brokers/master
```

Set the username and password to `admin`, and set Anonymous Access to `Y`.  Do the same for Slave One and Slave Two machines, except change the artemis setup command to:

```
./bin/artemis run brokers/slaveX
```

replacing `X` with your instance name.


6. For each instance, open the `amq-broker-7.3.0-bin/brokers/<instance name>/etc/bootstrap.xml` file and update the bind address to listen on all network addresses (`0.0.0.0`):

```
<!-- The web server is only bound to localhost by default -->
<web bind="http://0.0.0.0:8161" path="web">
    <app url="redhat-branding" war="redhat-branding.war"/>
    <app url="artemis-plugin" war="artemis-plugin.war"/>
    <app url="console" war="console.war"/>
</web>
```

7. For each instance, open the `amq-broker-7.3.0-bin/brokers/<instance name>/etc/jolokia-access.xml` file and update the CORS allow origin section to accept remote addresses:

```
        <allow-origin>*://*</allow-origin>
```
8. Copy and replace the `amq-broker-7.3.0-bin/brokers/<instance name>/etc/broker.xml` file with the corresponding `broker.xml` for each instance.  Make sure you replace the `m.m.m.m` with the Master Elastic IP, `s1.s1.s1.s1` with the Slave One Elastic IP, and `s2.s2.s2.s2` with the Slave Two Elastic IP in the `<connectors>` section:

```
<!-- Connectors -->

<connectors>
   <connector name="netty-connector">tcp://m.m.m.m:61616</connector>
   <!-- connector to the server1 (slave1) -->
   <connector name="server1-connector">tcp://s1.s1.s1.s1:61616</connector>
   <!-- connector to the server2 (slave2) -->
   <connector name="server2-connector">tcp://s2.s2.s2.s2:61616</connector>
</connectors>
```

9. Startup the Master instance first (`./amq-broker-7.3.0-bin/brokers/<instance name>/bin/artemis run`).  Wait for it to startup, then startup Slave One and Slave Two instances in the same manner.

### Testing the cluster

We can test failover between master / slave nodes using the following procedure:

1. On your local machine, open a shell and navigate to the directory where the legacy ActiveMQ client JAR is.  Execute the following command for the consumer (replacing m, s1 and s2 values with your corresponding Elastic IP's):

```
java -jar activemq-all-5.14.5.jar consumer --user admin --password admin --brokerUrl 'failover:(tcp://m.m.m.m:61616,tcp://s1.s1.s1.s1:61616,tcp://s2.s2.s2.s2:61616)' --destination queue://TEST
```

2. In a separate shell, do the same for the Producer, except use the following command:

```
java -jar activemq-all-5.14.5.jar producer --messageCount 1000 --user admin --password admin --brokerUrl 'failover:(tcp://m.m.m.m:61616,tcp://s1.s1.s1.s1:61616,tcp://s2.s2.s2.s2:61616)' --destination queue://TEST --sleep 100
```

Notice that both Producer and Consumer connect to the Master node and a message is sent every half second.  Experiment with failoving over between Master / Slave by killing each broker.  Notice how failover occurs using client-side loadbalancing.  There should be zero message loss.  You can verify this if both Producer and Consumer exist gracefully after sending / receiving 1000 messages.  Happy Testing!

![Testing](images/testing.png)
