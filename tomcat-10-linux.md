# Apache Tomcat 10 with EXPRESSCLUSTER X on Linux

## Table of Contents

* [Evaluation Environment](#evaluation-environment)
* [Installing OpenJDK](#installing-openjdk)
* [Setting up the cluster](#setting-up-the-cluster)
* [Setting up Apache Tomcat](#setting-up-apache-tomcat)
* [Modifying the configuration for Apache Tomcat](#modifying-the-configuration-for-apache-tomcat)
* [Test](#test)

## Evaluation Environment

* Red Hat Enterprise Linux 8.6
* OpenJDK 11.0.14.1.1-6
* Apache Tomcat 10.1.12
* EXPRESSCLUSTER X 5.1.1
  * The necessary licenses:
    * EXPRESSCLUSTER X
    * EXPRESSCLUSTER X Replicator
    * EXPRESSCLUSTER X Java Resource Agent

### Network Configuration

```
+---------- cluster ---------+
|                            |
|  [primary]-----[seoncdary] |
|                            |
+----------------------------+
```

* primary
  * IP address: 192.168.0.101
* secondary
  * IP address: 192.168.0.102

## Installing OpenJDK

```
# dnf install java-11-openjdk
# java -version
openjdk version "11.0.14.1" 2022-02-08 LTS
OpenJDK Runtime Environment 18.9 (build 11.0.14.1+1-LTS)
OpenJDK 64-Bit Server VM 18.9 (build 11.0.14.1+1-LTS, mixed mode, sharing)
```

## Setting up the cluster

### Installing EXPRESSCLUSTER X

```
# rpm -i expresscls-5.1.1-1.x86_64.rpm
```

### Configuring groups and monitors

* Groups
  * failover
    * fip (Floating IP resource)
      * IP Address: 192.168.0.110
    * md (Mirror disk resource)
      * Mount Point: /mnt/md1
      * Data Partition Dev. Name: /dev/md1/dp
      * Cluster Partition Dev. Name: /dev/md1/cp
      * File System: ext4
* Monitors
  * fipw1
  * mdnw1
  * mdw1
  * userw

### Starting the cluster

1. After completing the configuration, start the cluster.

```
# clpcl -s -a
```

2. Check whether the cluster is working fine.

```
# clpstat
 ========================  CLUSTER STATUS  ===========================
  Cluster : TomcatCluster
  <server>
   *server1 .........: Online           
      lankhb1        : Normal           Kernel Mode LAN Heartbeat
    server2 .........: Online           
      lankhb1        : Normal           Kernel Mode LAN Heartbeat
  <group>
    failover ........: Online           
      current        : server1
      fip            : Online           
      md             : Online           
  <monitor>
    fipw1            : Normal           
    mdnw1            : Normal           
    mdw1             : Normal           
    userw            : Normal           
 =====================================================================
```

## Setting up Apache Tomcat

### Installing Apache Tomcat

1. Download Apache Tomcat from https://tomcat.apache.org/ .

```
# curl -o apache-tomcat-10.1.12.tar.gz https://dlcdn.apache.org/tomcat/tomcat-10/v10.1.12/bin/apache-tomcat-10.1.12.tar.gz
```

2. Extract the tar ball and place the apache-tomcat directory wherever you like.

```
# tar xzf apache-tomcat-10.1.12.tar.gz
# mv apache-tomcat-10.1.12 /opt
```

### Arranging some directories under the apache-tomcat directory

**On the primary server**

1. Create a new directory on the mirror disk.  
   (Please make sure that the failover group is active on the primary server.)

```
# mkdir /mnt/md1/apache-tomcat
```

2. Move some directories from the apache-tomcat directory to the mirror disk.

```
# cd /opt/apache-tomcat-10.1.12
# mv bin conf webapps /mnt/md1/apache-tomcat
```

3. Create symbolic links to the moved directories.

```
# for d in bin conf webapps
> do
> ln -s /mnt/md1/apache-tomcat/$d $d
> done

# ls -l bin conf webapps
lrwxrwxrwx 1 root root 26 Aug 15 13:40 bin -> /mnt/md1/apache-tomcat/bin
lrwxrwxrwx 1 root root 27 Aug 15 13:40 conf -> /mnt/md1/apache-tomcat/conf
lrwxrwxrwx 1 root root 30 Aug 15 13:40 webapps -> /mnt/md1/apache-tomcat/webapps
```

4. Move the failover group to the secondary server for a next procedure.

```
# clpgrp -m
```

**On the secondary server**

1. Delete some directories from the apache-tomcat directory.

```
# cd /opt/apache-tomcat-10.1.12
# rm -fr bin conf webapps
```

2. Create symbolic links to the moved directories.

```
# for d in bin conf webapps
> do
> ln -s /mnt/md1/apache-tomcat/$d $d
> done
```

## Modifying the configuration for Apache Tomcat

1. Add "Java Installation Path" setting.
   * Cluster Properties
     * JVM Monitor tab
       * Java Installation Path: (e.g.) /usr/lib/jvm/jre-11-openjdk

2. Add an EXEC resource (exec1).
   * start.sh
```
#! /bin/sh
#***************************************
#*              start.sh               *
#***************************************

/opt/apache-tomcat-10.1.12/bin/startup.sh
exit $?
```
   * stop.sh
```
#! /bin/sh
#***************************************
#*               stop.sh               *
#***************************************

/opt/apache-tomcat-10.1.12/bin/shutdown.sh
exit $?
```

3. Add a JVM monitor resource.
   * Monitor(special) tab
     * Target: Tomcat
     * JVM Type: OpenJDK
     * Identifier: tomcat
     * Connection Port: 1099
   * Recovery Action tab
     * Recovery Action: Executing failover to the recovery target
     * Recovery Target: exec1

4. Create a new setenv.sh file in /opt/apache-tomcat-10.1.12/bin directory.
   * The setenv.sh should have the following contents **written in ONE line.**
   * The value of jmxremote.port should be the same value of "Connection Port" in the JVM monitor.

```
CATALINA_OPTS="${CATALINA_OPTS} -Dcom.sun.management.jmxremote.port=1099 -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false"
```

## Test

1. Active the exec resource and then check whether the cluster is working fine.

```
# clpstat
 ========================  CLUSTER STATUS  ===========================
  Cluster : TomcatCluster 
  <server>
   *server1 .........: Online           
      lankhb1        : Normal           Kernel Mode LAN Heartbeat
    server2 .........: Online           
      lankhb1        : Normal           Kernel Mode LAN Heartbeat
  <group>
    failover ........: Online           
      current        : server1
      exec           : Online           
      fip            : Online           
      md             : Online           
  <monitor>
    fipw1            : Normal           
    jraw             : Normal           
    mdnw1            : Normal           
    mdw1             : Normal           
    userw            : Normal           
 =====================================================================
```

2. Access the Apache Tomcat via the floagint IP. (e.g. `http://192.168.0.110:8080`)

3. Stop the tomcat process intentionally and then check whether the recovery action (i.e. failover) will be executed.

```
# /opt/apache-tomcat-10.1.12/bin/shutdown.sh

(wait for an error detection by jraw)

# clpstat
 ========================  CLUSTER STATUS  ===========================
  Cluster : TomcatCluster 
  <server>
   *server1 .........: Online           
      lankhb1        : Normal           Kernel Mode LAN Heartbeat
    server2 .........: Online           
      lankhb1        : Normal           Kernel Mode LAN Heartbeat
  <group>
    failover ........: Online           
      current        : server2     (<- moved to the secondary srv)
      exec           : Online           
      fip            : Online           
      md             : Online           
  <monitor>
    fipw1            : Normal           
    jraw             : Normal           
    mdnw1            : Normal           
    mdw1             : Normal           
    userw            : Normal           
 =====================================================================
```
