# AMQ 7.x Master / Slave Replicated Cluster on EC2


This project demonstrates how to setup a three-node AMQ 7 Master / Slave cluster (using shared-nothing replication) within a single Amazon EC2 Availability Zone (AZ).  Please note that although this architecture is limited to a single AZ only.  Although it *may* work cross-AZ, it is not recommended nor supported.

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
./bin/artemis run brokers/master
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

### Test LDAP connectivity with your broker cluster

Although we want to enforce RBAC at the IC router level, it's important to test out LDAP connectivity using our users / groups.  For this exercise, we'll use a single standalone broker instead of the cluster.

1. Create a new broker instance using the following command:

```
./bin/artemis create instances/adbroker
```

Use admin/admin for credentials and type 'Y' for anonymous access

2. Replace the `etc/broker.xml`, `etc/bootstrap.xml` and `etc/login.config` files with those contained in this project (found in `/conf/broker/ldap`)

3. Update the HawtIO role in artemis.profile with `-Dhawtio.role=admins`

4. Startup the broker using `bin/artemis run`

5. Test login to the HawtIO console using both your admin and user credentials.  Only the admin user should be authorized to enter the console.

6. Test sending messages using a known username e.g. jdoe:

```
./bin/artemis producer --message-count 10 --url "tcp://127.0.0.1:61616" --destination queue://defQueue --user jdoe --password sunflower
```

7. Test consuming messages using a known username e.g. jdoe:

```
./artemis consumer --message-count 10 --url "tcp://127.0.0.1:61616" --destination queue://defQueue --user jdoe --password sunflower
```

### Setup Interconnect Router for SSL Client connections

This procedure sets up PLAIN SSL for clients connecting via the router.  The router contains autolinks to a couple of static queues (queue.foo and queue.grs).  The router will loadbalance between the active/passive master/slave pair, depending on which is active.

1. Install the Interconnect Router on RHEL 7 as per the [instructions](https://access.redhat.com/documentation/en-us/red_hat_jboss_amq/7.0/html/using_amq_interconnect/installation)

2. Follow the instructions to setup an SSL profile from the [documentation](https://access.redhat.com/documentation/en-us/red_hat_jboss_amq/7.0/html/using_amq_interconnect/security#setting_up_ssl_for_encryption_and_authentication).  I used openssl to generate my cacerts and keystores: `openssl req -newkey rsa:2048 -nodes -keyout key.pem -x509 -days 365 -out certificate.pem`.

3. Copy the `/conf/ic/qdrouterd.conf` to your RHEL server.

4. Update the qdrouterd.conf sslProfile section to include the correct path to your keys and keystores created in step 2.

5. Startup your master/slave brokers.  You should see the following in the router log indicating the connection to the master broker is active:

```
Fri Nov 10 13:17:52 2017 ROUTER_CORE (info) Auto Link Activated 'autoLink/0' on connection master
Fri Nov 10 13:17:52 2017 ROUTER_CORE (info) Auto Link Activated 'autoLink/1' on connection master
Fri Nov 10 13:17:52 2017 ROUTER_CORE (info) Auto Link Activated 'autoLink/2' on connection master
Fri Nov 10 13:17:52 2017 ROUTER_CORE (info) Auto Link Activated 'autoLink/3' on connection master
```

6. Send a test message to the master node by issuing the following command:

```
qsend amqps://127.0.0.1:5672/queue.foo -m Master
```

7. Login to the Master node HawtIO app.  You should see the message in the queue.foo by browsing.

8.  Failover to the Slave broker by killing the Master.  The following should appear in the router logs:

```
Fri Nov 10 13:21:34 2017 ROUTER_CORE (info) Auto Link Deactivated 'autoLink/0' on connection master
Fri Nov 10 13:21:34 2017 ROUTER_CORE (info) Auto Link Deactivated 'autoLink/1' on connection master
Fri Nov 10 13:21:34 2017 ROUTER_CORE (info) Auto Link Deactivated 'autoLink/2' on connection master
Fri Nov 10 13:21:34 2017 ROUTER_CORE (info) Auto Link Deactivated 'autoLink/3' on connection master
Fri Nov 10 13:21:34 2017 ROUTER_CORE (info) Auto Link Activated 'autoLink/4' on connection slave
Fri Nov 10 13:21:34 2017 ROUTER_CORE (info) Auto Link Activated 'autoLink/5' on connection slave
Fri Nov 10 13:21:34 2017 ROUTER_CORE (info) Auto Link Activated 'autoLink/6' on connection slave
Fri Nov 10 13:21:34 2017 ROUTER_CORE (info) Auto Link Activated 'autoLink/7' on connection slave
```

9. Send a test message to the slave node by issuing the following command:

```
qsend amqps://127.0.0.1:5672/queue.foo -m Slave
```

### Setup LDAP connection to IC Router

1. Install the necessary cyrus-sasl libraries required by the router and also set up the listener.  The
`cyrus-sasl` and `cyrus-sasl-plain` libraries need to be installed (`yum install cyrus-sasl cyrus-sasl-plain`). The client sends the user name and password in clear to the Router. Hence, in a production environment, this client-router communication needs to be done over a TLS connection.

Setup the following listener in the router's config file (this is the config file you use to start the router) - port number can be to your choosing:
```
listener {
    addr: 0.0.0.0
    port: 15677
    role: normal
    authenticatePeer: yes
    saslMechanisms: PLAIN
}
```
Notice that this sets up the Router listener to use PLAIN as the the only mechanism used to communicate with the client. PLAIN is used because the user name and password needs to be passed in clear to the PAM authentication module.

2. Configure Dispatch Router to use a program called saslauthd to connect to SSSD via PAM (Pluggable Authentication Modules).
The sasl config file is usually found in the /etc/sasl2/qdrouterd.conf file. Add the following properties to this file:
```
pwcheck_method: saslauthd
auxprop_plugin:pam
mech_list: PLAIN
```
Notice that the `pwcheck_method` is set to `saslauthd` which is a program that is installed as part of the `cyrus-sasl` library installation (`yum install cyrus-sasl`). `saslauthd` is used to supply the user name and password to the pam module. Notice here that the `auxprop_plugin` is set to pam which instructs cyrus-sasl to enable authentication using PAM via saslauthd. The `mech_list` is set to `PLAIN`  to match the mech_list in step 1.

Make sure that the `/etc/sysconfig/saslauthd` file contains
`MECH=pam`

This enables saslauthd to use PAM.

3. Configure PAM
Every application has to have its own config file in the `/etc/pam.d/` folder. The config file for Qpid Dispatch is called amqp. Open or create the `/etc/pam.d/amqp` file and add the following to it -
`#%PAM-1.0
auth    required  pam_securetty.so
auth    required pam_nologin.so
account required pam_unix.so
session required pam_unix.so
`
What each of the above lines means is explained in detail in Section 2.2 [here](docs/pam-step-3.pdf)
The above PAM configuration is not production grade but works in my situation.

4. Configure PAM service on SSSD - This instructs PAM to use SSSD to retrieve user information and is dealt with in detail in Section 30.3.2 [here](docs/pam-step-4.pdf) (I followed all steps in this link)
(SSSD is already installed on RHEL 7 machines)

5. Install Redhat IdM (Active Directory(AD) and any LDAP server can be used instead of Redhat IdM) in Section 2.3 [here](docs/pam-step-5.pdf).
Skip this step if you are not using IdM

6. Ask SSSD to discover AD or IdM services `realm discover test.example.com` - This instructs SSSD to discover a directory service running on the host test.example.com.
Use - `realm discover --server-software=active-directory test.example.com` - to discovery AD
If the discovery is successful, to join the system SSSD to an identity domain, use the realm join command and specify the domain name:
```
realm join test.example.com
```
SSSD will successfully join the realm. Test the whole setup with the testsaslauthd program that comes as part of the cyrus-sasl installation
```
[root@amq01 /]# testsaslauthd -u test -p test -r LAB.ENG.RDU2.REDHAT.COM -s amqp
0: OK "Success."
```
If you don't get "OK", you are in trouble.

To troubleshoot watch the output of `journalctl -f` as you run `testsaslauthd`

7. In summary, now you have enabled SASL in Qpid Dispatch Router to talk to PAM which in turn talks to SSSD which in turn can talk to any directory service like AD, LDAP or IdM.

8. Finally start the router and run qdstat

`reset; PN_TRACE_FRM=1 qdstat -b amqps://127.0.0.1:15677 -c --sasl-mechanisms=PLAIN --sasl-username=max@LAB.ENG.RDU2.REDHAT.COM --sasl-password=Abcd1234`

Note here again the the saslMechanisms is set to PLAIN and the realm (LAB.ENG.RDU2.REDHAT.COM) is included as part of the user name.# amq7-replicated-cluster-ec2
