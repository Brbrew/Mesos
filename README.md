# Mesosphere Installation Notes
This is a simple example of 4 node (2x [Masters + Slaves] + 2x [Dedicated Slaves]) instance using using RHEL 7 
System V is installed by default, Upstart is not present. 
Keep in mind everything here is running as ROOT for example purposes.
Specific configuration and security settings are beyond the scope of this document.

> Master1: 192.168.1.101 node2 MYID2  
> Master2: 192.168.1.100 node1 MYID1  
> Salve3:	 192.168.1.102  node3  
> Salve3:	 192.168.1.103 node4 

Much of this is based on the [Mesosphere Advanced Course](https://open.mesosphere.com/advanced-course/). 

## Zookeeper
First, set up Zookeeper on all **MASTER** servers of the zookeeper cluster.

Install Zookeeper and the Zookeeper server package by pointing to the RPM repository for ZooKeeper:

* Add CDH5 Repo:
	* `sudo rpm -Uvh https://archive.cloudera.com/cdh5/one-click-install/redhat/7/x86_64/cloudera-cdh-5-0.x86_64.rpm`	
	
* Install Zookeeper and Zookeeper-Server
	* `sudo -i` - Interactive mode
	* `$ yum -y install zookeeper zookeeper-server`
		
* Initialize and start Zookeeper
	* Note: MyID **MUST** be *unique* for each instance. So for the first MASTER node, myid=1; the second myid=2; etc. 
	*  `$ sudo -u zookeeper zookeeper-server-initialize --myid=1` (Assumes first node)
	
* Check config file of Zookeeper	
	* `$ cat /etc/zookeeper/conf/zoo.cfg`
	* Example [/etc/zookeeper/conf/zoo.cfg](/etc/zookeeper/conf/zoo.cfg)
	
	* The output should looks something like this:
	 
			# The maximum numer of clients
			maxClientCnxns=50 
			# The number of milliseconds of each tick 
			tickTime=2000 
			# The number of ticks that the initial synchronization phase can take 
			intLimit=10  
			# The number of ticks that can pass between sending a request and getting an acknowledgement 
			syncLimit=5  
			# the directory where the snapshot is stored. 
			dataDir=/var/lib/zookeeper  
			# the port at which the clients will connect 
			clientPort=2181	  
			# the directory where the transaction logs are stored. 
			dataLogDir=/var/lib/zookeeper  
			#List of servers (typically this is not added by default):
			server.1=192.168.1.101:2888:3888
			server.2=192.168.1.100:2888:3888
			
	* Most important thing to note is **adding the list of servers** (i.e. server.1, server.2, etc.)
	
* Start Zookeeper
	*  `$ zookeeper-server start`  	
	*  You will need to add the Zookeeper start service to /init or init.d to make sure the service starts on reboot.
	
* Test installation using zkCLI
	* `/usr/lib/zookeeper/bin/zkCli.sh`
	* `[zk: localhost:2181(CONNECTED) 0] create /test 1`
	* `[zk: localhost:2181(CONNECTED) 0] get /test`
	* `[zk: localhost:2181(CONNECTED) 0] create /test 2`
	* `[zk: localhost:2181(CONNECTED) 0] get /test`
	* `[zk: localhost:2181(CONNECTED) 0] delete /test`
	* `[zk: localhost:2181(CONNECTED) 0] quit`
	
* For a multi-node configuration, validate leader and follower (run on each server):
* `$ echo stat | nc localhost 2181 | grep Mode`
	> Mode: follower
	> Mode: leader
	
### Checks
Before moving on, double check the following:

* Is Zookeeper installed on **ALL** *MASTER* nodes?
* Has that been validated via zkCLI?
* Has each machine rebooted and validated that the Zookeeper service correctly started?
* Has the leader been validated?

### Further Reading
* [Cloudera CDH5 Notes](http://www.cloudera.com/documentation/enterprise/5-5-x/topics/cdh_ig_cdh5_install.html)
* [Cloudera Zookeeper Guide](http://www.cloudera.com/documentation/enterprise/5-5-x/topics/cdh_ig_zookeeper_package_install.html#topic_21_3_3)
* [Apache Zookeeper Admin](http://zookeeper.apache.org/doc/r3.3.2/zookeeperAdmin.html)
* [Apache Getting started](https://zookeeper.apache.org/doc/trunk/zookeeperStarted.html)
	
## Install Mesos 
Install Mesos packages. Mesos needs to be installed on **ALL** servers, both master and agents.
Depending on the purpose of the node, some will only need to run the mesos-master service, the mesos-agent service, or both.
Keep in mind that most frameworks will need to be ran on the agent nodes. Only MESOS, ZOOKEEPER, and MARATHON should be installed on MASTER nodes.
All other frameworks need to be installed on the agent nodes and ran via MARATHON. 

* Add Mesosphere packages
	* `$ sudo rpm -Uvh http://repos.mesosphere.com/el/7/noarch/RPMS/mesosphere-el-repo-7-1.noarch.rpm`
* Install Mesos
	* `$ sudo yum -y install mesos`

### Configure instances
For vSphere clusters, DNS registration occurs only locally. 
So while the servers recognize each other's DNS (via */etc/hosts*), those servers may not be registered with the global DNS server. 
For that reason, we force the hostname of each server to be the IP by default.

#### Master Node 1 (Primary)

* Set hostname
	* `echo 192.168.1.100 | sudo tee /etc/mesos-master/hostname`
	* Example [/etc/mesos-master/hostname](/etc/mesos-master/hostname)

* Set IP	
	* `echo 192.168.1.100 | sudo tee /etc/mesos-master/ip`
	* Example [/etc/mesos-master/ip](/etc/mesos-master/ip)
		
* Set quorum	
	* `echo 1 | sudo tee /etc/mesos-master/quorum`
	* Example [/etc/mesos-master/quorum](/etc/mesos-master/quorum)

* Set clustername (Not necessary to name the cluster, but why not right?)
	* `echo BrewCluster | sudo tee /etc/mesos-master/cluster`
	* Example [/etc/mesos-master/cluster](/etc/mesos-master/cluster)
	
* **IMPORTANT:** Since this a 2 MASTER node set up, the QUORUM is set to one. 
	> A cluster can sustain failures of half the nodes (rounding up) minus one. So for example, a seven node cluster can sustain three node failures, so in that case the quorum would be 4. In our case, since we only have a 2 node cluster, the quorum is 1. 
	* `echo 1 | sudo tee /etc/mesos-master/quorum`
	* `echo zk://192.168.1.101:2181,192.168.1.100:2181/mesos | sudo tee /etc/mesos/zk`

#### Master Node 2 (Secondary)
Same as above and for each MASTER node.

* Set hostname
	* `echo 192.168.1.100 | sudo tee /etc/mesos-master/hostname`
	* Example [/etc/mesos-master/hostname](/etc/mesos-master/hostname)

* Set IP	
	* `echo 192.168.1.100 | sudo tee /etc/mesos-master/ip`
	* Example [/etc/mesos-master/ip](/etc/mesos-master/ip)
	
* Set clustername (optional)	
	* `echo BrewCluster | sudo tee /etc/mesos-master/cluster`
	* Example [/etc/mesos-master/cluster](/etc/mesos-master/cluster)
	
* Set quorum	
	* `echo 1 | sudo tee /etc/mesos-master/quorum`
	* Example [/etc/mesos-master/quorum](/etc/mesos-master/quorum)

* Be sure to list ALL MASTER nodes in your zk file:
	* `echo zk://192.168.1.101:2181,192.168.1.100:2181/mesos | sudo tee /etc/mesos/zk`
	* Example [/etc/mesos/zk](/etc/mesos/zk)

#### Start and Validate instances

* Start Instances
	* `$ sudo systemctl start mesos-master`		
	* `$ sudo systemctl start mesos-agent`	
		
* Check services are running (run on both nodes)
	* `$ netstat -nlp | grep mesos`	
	> tcp        0      0 192.168.1.101:5050     0.0.0.0:*               LISTEN      4263/mesos-master <br/>
	> tcp        0      0 0.0.0.0:5051            0.0.0.0:*               LISTEN      4310/mesos-agent

* Check mesos Web GUI: 
	* `http://192.168.1.101:5050`
* Under "Frameworks" in the Web GUI, you should see a marthon framework
* Test
	* `$ export MASTER=$(mesos-resolve `cat /etc/mesos/zk` 2>/dev/null)` -- Create local variable; output of ZK (dynamic)
	* `$ mesos-execute --master=$MASTER --name="cluster-test" --command="sleep 40"`
	* `$ mesos-execute --master="192.168.1.101:5050" --name="test-exec" --command="sleep 10"`
	
#### Specific Slave Node Configuration
Note: Slaves DO NOT need Zookeeper installed

* Add Mesosphere packages
	* `$ sudo rpm -Uvh http://repos.mesosphere.com/el/7/noarch/RPMS/mesosphere-el-repo-7-1.noarch.rpm`
	* Example [/etc/yum.repos.d/mesosphere.repo](/etc/yum.repos.d/mesosphere.repo)

* Install Mesos
	* `$ sudo yum -y install mesos`

* Add master server list to mesos/zk
	* `$ echo zk://192.168.1.101:2181,192.168.1.100:2181/mesos | sudo tee /etc/mesos/zk`
	* Example [/etc/mesos/zk](/etc/mesos/zk)

* Install agent frameworks as needed
	* Chronos/HDFS/Docker/Etc.
	* NOT Marathon; Marthon orchestrates frameworks and long running processes from MASTER nodes ONLY.

### Java Troubleshooting (if needed)
* Check the syslogs:
	* `$ cat /var/log/messages | grep mesos`
	* `$ cat /var/log/mesos` 
* Look for any Java error messages, you may find you need to upgrade the version. Java versions are as follows:

		J2SE 8 = 52, 
		J2SE 7 = 51, 
		J2SE 6.0 = 50, 
		J2SE 5.0 = 49, 
		JDK 1.4 = 48, 
		JDK 1.3 = 47, 
		JDK 1.2 = 46, 
		JDK 1.1 = 45
	 
* Validate and check version running
	* `$ alternatives --install /usr/bin/java java /opt/jdk1.8.0_91/bin/java 2`
	* `$ alternatives --config java`
	
* The output of the java alternatives should look like the below:

			There are 4 programs which provide 'java'. 
			Selection    Command 
			----------------------------------------------- 
			*  1        /usr/lib/jvm/java-1.7.0-openjdk-1.7.0.95-2.6.4.0.el7_2.x86_64/jre/bin/java
			2           /usr/lib/jvm/java-1.7.0-oracle-1.7.0.95-1jpp.2.el7.x86_64/jre/bin/java
			3           /usr/lib/jvm/java-1.8.0-ibm-1.8.0.3.0-1jpp.1.el7.x86_64/jre/bin/java 
			+ 4         /opt/jdk1.8.0_91/bin/java 
			Enter to keep the current selection[+], or type selection number:
	   
* In this specific case, choose (4).
* If there is a need to upgrade, run the following.
* Download latest Java package and install to /opt
	* `$ cd /opt/`
* Download
	* `$ wget --no-cookies --no-check-certificate --header "Cookie: gpw_e24=http%3A%2F%2Fwww.oracle.com%2F; oraclelicense=accept-securebackup-cookie" "http://download.oracle.com/otn-pub/java/jdk/8u91-b14/jdk-8u91-linux-x64.tar.gz"`
* Install to /opt
	* `$ tar xzf jdk-8u91-linux-x64.tar.gz`
	* `$ cd /opt/jdk1.8.0_91/`
* Re-run `$ alternatives --config java` and select correct version

### Start Mesos
* Start services 
	* `$ systemctl start mesos-master`
	* `$ systemctl start mesos-agent`

### Mesos Slave Troubleshooting
 * Export master and hostname to local var
	 * `$ export MESOS_MASTER=192.168.1.100:5050`
	 * `$ export MESOS_HOSTNAME=192.168.1.100`
 * If you see the following error:
 	> mesos-master.localhostname.invalid-user
 * Run the following:
	 * Step 1: `$ rm -f /tmp/mesos/meta/agents/latest`
		 * This ensures agent doesn't recover old live executors.
	 * Step 2: Restart the agent. 	 
	 
### Important Note - *EVERY TIME* the agent is shut down, be sure to run the following:
 * `$ rm -f /tmp/mesos/meta/agents/latest`
 * This will clear the agent cash and reset the agent node

### Further Reading
* [Mesos Framework Dev Guide](http://mesos.apache.org/documentation/latest/app-framework-development-guide/)

## Install and Configure Marathon
Marathon is a framework and container orchestration platform for Apache Mesos with a nice Web UI. Marathon is designed to run long running services (like frameworks or service based Docker containers). Specific ETL jobs need to be ran of a scheduler like Chronos; however since Chronos is a framework, Chronos itself would be kicked off from Marathon.

* Install Marathon on all MASTER nodes
	* `$ sudo yum -y install marathon`

### Start Marathon
* Start services 
	* `$ systemctl start marathon`

#### Validate Instance
* Again, double check the framework is present:
	* http://192.168.1.100:5050/#/frameworks
* If still having issues, validate the ports are listening:
	* `$ netstat -nlp | grep :5050` 
		> tcp        0      0 192.168.1.100:5050      0.0.0.0:*               LISTEN      9490/mesos-master
	* `$ netstat -nlp | grep :8080`
		> tcp6       0      0 :::8080                 :::*                    LISTEN      9536/java

#### Testing Marathon GUI

* Create a new application called "test"
	* `python -m SimpleHTTPServer` 
	* The JSON config script should look something like this:
	* Example [examples/simplehttp.json](/examples/simplehttp.json)
		
			{
			  "id": "/test",
			  "cmd": "python -m SimpleHTTPServer",
			  "cpus": 1,
			  "mem": 32,
			  "disk": 0,
			  "instances": 1,
			  "portDefinitions": [
			    {
			      "port": 10000,
			      "protocol": "tcp",
			      "labels": {}
			    }
			  ]
			}
			
* Validate python SimpleHTTPServer web server is running
	* `$ netstat -nlp | grep :8000`
	
* Curl to test server
	* `$ curl http://192.168.1.100:8000/`
	* `$ curl http://192.168.1.100:8000/stderr`
	* `$ curl http://192.168.1.100:8000/stdout`
	
### Further Reading

* [Mesosphere Marathon](https://mesosphere.github.io/marathon/)
* [Documentation](https://mesosphere.github.io/marathon/docs/)

## Install Docker (RHEL)
Note: Run the following on each agent node (including masters if they are running agent instances).

* Validate kernel version, requires 3.10
	* `$ uname -r`
		> 3.10.0-229.el7.x86_64
		
* Add Docker repo:
* Example [/etc/yum.repos.d/docker.repo](/etc/yum.repos.d/docker.repo)

		$ cat >/etc/yum.repos.d/docker.repo <<-EOF
		[dockerrepo]
		name=Docker Repository
		baseurl=https://yum.dockerproject.org/repo/main/centos/7
		enabled=1
		gpgcheck=1
		gpgkey=https://yum.dockerproject.org/gpg
		EOF
		
* Install docker
	* `$ sudo yum -y install docker-engine `
* Start Docker
	* `$ sudo systemctl start docker` 
* Validate Docker
	* `$ sudo docker run hello-world`
* Update agent configuration to specify the use of the Docker containerizer 
	* `$ sudo echo 'docker,mesos' > /etc/mesos-agent/containerizers`
	* Example [/etc/mesos-agent/containerizers](/etc/mesos-agent/containerizers)
	
* Below step is not 100% necessary (only if you want Marathon to pull the images) 
	* `$ echo '5mins' > /etc/mesos-agent/executor_registration_timeout`
	* Example [/etc/mesos-agent/executor_registration_timeout](/etc/mesos-agent/executor_registration_timeout)
	
* Restart mesos-agent
	* `$ sudo systemctl restart mesos-agent`	

### Validate boot processes
* `$ chkconfig docker on`
* `$ chkconfig marathon on`
* `$ chkconfig mesos-master on`
* `$ chkconfig mesos-agent on`

### Further Reading
* [Docker Install](https://docs.docker.com/v1.8/installation/rhel/)

## Setup Mesos DNS
Mesos-DNS supports service discovery in Apache Mesos clusters. It allows applications and services running on Mesos to find each other through the domain name system (DNS), similarly to how services discover each other throughout the Internet. Applications launched by Marathon or Aurora are assigned names like search.marathon.mesos or log-aggregator.aurora.mesos. Mesos-DNS translates these names to the IP address and port on the machine currently running each application.

Why this is important. 
For microservices and general cluster load-balancing, multiple applications can run under a different port. 
For example, if we scale Python's SimpleHTTP and assign it a random port through the $PORT variable, a user hitting say Port 8000 will not necessarily be routed to the appropriate port. Mesos DNS automatically sets up that mapping and <b>Service Discovery</b>

For more, see [NGINX Excellent Explanation](https://www.nginx.com/blog/service-discovery-in-a-microservices-architecture/)

#### Installation
Install on ALL agent nodes.

* Install GoLang
	* `$  yum install golang`
* Install Bind-utils
	* `$  yum install bind-utils`			
		> Bind-Utils contains a collection of utilities for querying DNS (Domain Name Service) name servers to find out information about Internet hosts. These tools will provide you with the IP addresses for given host names, as well as other information about registered domains and network addresses. You should install bind-utils if you need to get information from DNS name servers.
* Install GIT
	* `$  yum install git-all`
		> Note: The package git-svn allows git access to subversion repos
* Install Device Mapper
	* ` yum install device-mapper-event-libs `

### Build Mesos DNS
* General note:					
	> Since Mesos doesn't have the idea of dependencies, if applications inside of your cluster depend on Mesos DNS, and Mesos DNS hasn't yet come up, you can run into a bootstrapping problem.
	> DCOS specifically doesn't do that because Mesos agents themselves rely on Mesos-DNS.

* More: [Mesos-DNS Service Discovery](https://mesosphere.com/blog/2015/01/21/mesos-dns-service-discovery/)

* How to Install:
	* `$ mkdir ~/go`
	* `$ export GOPATH=$HOME/go`
	* `$ export PATH=$PATH:$GOPATH/bin`
	* `$ go get github.com/tools/godep`
	* `$ go get github.com/mesosphere/mesos-dns`
	* `$ cd $GOPATH/src/github.com/mesosphere/mesos-dns`
	* `$ go build .` (Don't forget the dot at the end!)
	
* Add local IP to resolv.conf
	* `$ nano /etc/resolv.conf`
	* Example [/etc/resolv.conf](/etc/resolv.conf)
	
* Add the following lines:	 
	* IMPORTANT!!!! - Add nameservers to the ***BEGINNING*** of /etc/resolv.conf so that it becomes the primary DNS router
 > To allow Mesos tasks to use Mesos-DNS as the primary DNS server, you must edit the file /etc/resolv.conf in every agent and add a new nameserver. For instance, if mesos-dns runs on the server with IP address 10.181.64.13, you should add the line nameserver 10.181.64.13 at the ***beginning*** of /etc/resolv.conf on every agent node. This can be achieve by running:

		nameserver 192.168.1.101
		nameserver 192.168.1.100	
		nameserver 192.168.1.102  
		nameserver 192.168.1.103	
		search mydomain.com 127.0.0.1
		nameserver 8.8.8.8
		nameserver 4.4.4.4
		

* Change config.json
	* Copy sample file provided
		* `$ cd $GOPATH/src/github.com/mesosphere/mesos-dns`
		* `$ cp config.json.sample config.json`
		
	* Adjust json file to look like this
	* Example [/etc/mesos/mesos_dns_config.json](/etc/mesos/mesos_dns_config.json)
	
			{
			  "zk": "zk://192.168.1.100:2181,192.168.1.101:2181/mesos",
			  "masters": ["192.168.1.100:5050","192.168.1.101:5050"],
			  "refreshSeconds": 60,
			  "ttl": 60,
			  "domain": "mesos",
			  "port": 53,
			  "resolvers": ["8.8.8.8","4.4.4.4","192.168.1.100","192.168.1.101"],
			  "timeout":5,
			  "httpon": true,
			  "dnson": true,
			  "httpport": 8123,
			  "externalon": true,
			  "listener": "0.0.0.0",
			  "SOAMname": "ns1.mesos",
			  "SOARname": "root.ns1.mesos",
			  "SOARefresh": 60,
			  "SOARetry":   600,
			  "SOAExpire":  86400,
			  "SOAMinttl": 60
			
			}

	* More:
		* [Mesos-DNS Config Parameters](https://mesosphere.github.io/mesos-dns/docs/configuration-parameters.html)
	* Validate locally
	* `~/go/src/github.com/mesosphere/mesos-dns/mesos-dns -v=1 -config=~/go/src/github.com/mesosphere/mesos-dns/config.json`
	* Or more simply (if in the dir): `mesos-dns -config=config.json &`
			 
			dig @<hostname> 
			mesos.master
			master0.mesos.
			test.marathon.agent.mesos.
			curl -X GET http://192.168.1.101:8123/v1/hosts/master.mesos -H "Content-type: application/json" 
			 
			zk://192.168.1.101:2181/mesos
			192.168.1.101:5050 


##### IMPORTANT NOTE:
* Copy to `/etc/mesos` so Marathon can find config file easily
* `cp ~/go/src/github.com/mesosphere/mesos-dns/config.json /etc/mesos/mesos_dns_config.json`
* Be sure to do this on ***ALL*** agent nodes
	
* Create a new app in Marathon with same command
* The JSON script for the Marathon app would look like this
* Example [examples/mesos-dns.json](examples/mesos-dns.json)
	
		{
		  "id": "mesos-dns",
		  "cmd": "~/go/src/github.com/mesosphere/mesos-dns/mesos-dns -v=1 -config=/etc/mesos/mesos_dns_config.json",
		  "cpus": .11,
		  "mem": 32,
		  "disk": 0,
		  "instances": 1			  
		}

* To have MESOS-DNS run on a specific node:
	* Add a constraint - `hostname:CLUSTER:192.168.1.100` 
		
### Further Reading
* [Mesos-DNS Reference](https://mesosphere.github.io/mesos-dns/docs/tutorial-forward.html)
* No space left on device? Try this:
	* [No Space Left](https://support.mesosphere.com/hc/en-us/articles/205683003--No-space-left-on-device-error-on-Mesos)		
* [Intro to Mesosphere](https://www.digitalocean.com/community/tutorials/an-introduction-to-mesosphere)
* [Mesos DNS Naming](https://mesosphere.github.io/mesos-dns/docs/naming.html)
* [Mesos DNS FAQ](https://mesosphere.github.io/mesos-dns/docs/faq.html)
* [Mesos DNS Troubleshooting](https://docs.mesosphere.com/usage/service-discovery/mesos-dns/faq-troubleshooting/)	
* [Mesos DNS Server](http://cookingdevops.blogspot.com/2015/11/mesos-dns-server.html)

## Install Chronos
Chronos is a basic scheduling system. The base setup is fairly easy once Zookeeper, Mesos, and Marthon are setup. Chronos (as all frameworks) can be ran without Marathon.

* `$ sudo yum install chronos`
* Launch through ***Marathon*** 
	* `/usr/bin/chronos`
* TODO: Config clustering
	* [Chronos Configuration](https://mesos.github.io/chronos/docs/configuration.html)

## Install HDFS
The Hadoop Distributed File System (HDFS) is a distributed file system that several systems (like Hadoop) sit on top of. For resource negotiation, typically HDFS utilizes [Apache YARN](https://hadoop.apache.org/docs/r2.7.1/hadoop-yarn/hadoop-yarn-site/YARN.html) which stands for Yet Another Resource Negotiator. Here, instead of YARN, we are putting HDFS on top of MESOS. Currently there is an incubator project tying together YARN and MESOS called [Myriad](http://myriad.incubator.apache.org/).  

The first step in getting Hadoop up and running is to install HDFS. The current version from MESOS-HDFS: 
> Hadoop 2.5.0 | Cloudera's CDH 5.3.1

Please keep in mind the total allocated resources for MESOS. You will see declined offers from the JournalNodes if there are not enough available resources. See Troubleshooting below for more info.

### Build MESOS-HDFS
The awesome part is that you only need to do this on 1 node. It will automatically replicate the package across all other nodes.
 
* Make sure GIT is installed
	* `yum install GIT`
* Create directory for building hdfs-mesos project
	* Note: Assuming current pwd is `/root` as `root` 
	* `$ mkdir hdfs-mesos`
	* `$ cd hdfs-mesos`
* Clone GIT copy to `~/hdfs-mesos`
	* `git clone https://github.com/mesosphere/hdfs.git`
* Change to GET directory
	* `$ cd hdfs`
* Install JAVA dev tools for current version of JAVA 
	* Check using `$ alternatives --config java`
	* `$ yum install java-1.8.0-ibm-devel`
	* Validate current version of Java matches
	* `$ alternatives --config java`
* Set $JAVA_HOME
	* Again, validate directory with `$ alternatives --config java`
	* `export JAVA_HOME=/usr/lib/jvm/java-1.8.0-ibm-1.8.0.3.0-1jpp.1.el7.x86_64/jre`
* Build project
	* Make sure PWD is `/root/hdfs-mesos/hdfs`
	* `$ ./bin/build-hdfs` 
	* Could take time depending on # of processors
* Copy and install project to /opt
	* `$ mkdir /opt/hdfs`
	* Make sure PWD is `/root/hdfs-mesos/hdfs/build`
		* `$ cd /build`
	* Copy tarball to /opt/hdfs
		* `$ cp hdfs-mesos-0.1.5.tgz /opt/hdfs`
	* Change to /opt dir
		* `$ cd /opt/hdfs`
	* Extract package
		* `$ tar zxvf hdfs-mesos-0.1.5.tgz`

### Update etc/hosts
* Adjust `/etc/hosts` to assign appropriate host names to all nodes on the cluster:
	* Example [/etc/hosts](/etc/hosts)

			127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
			::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
			8.8.8.8        mydomain.com
			192.168.1.101  node2.mydomain.com node2
			192.168.1.100  node1.mydomain.com node1
			192.168.1.102   node3.mydomain.com node3 
			192.168.1.103  node4.mydomain.com node4


### Configure and run 
* Update configuration as needed
	* `$ /opt/hdfs/hdfs-mesos-0.1.5/etc/hadoop/mesos-site.xml`
	* Example [hdfs-mesos/etc/hadoop/mesos-site.xml](hdfs-mesos/etc/hadoop/mesos-site.xml)

* Run
	* TODO: Execute from Marathon
	* Note: Needs to be ran in the directory
	*  `$ cd /opt/hdfs/hdfs-mesos-0.1.5`
	*  `$ ./bin/hdfs-mesos` 

### Using HDFS 
Note: You may need to set JAVA_HOME here if not already
* Should show nothing for starters
* `$ hadoop fs -ls hdfs://hdfs/`
> /bin/hadoop: line 2: /opt/mesosphere/hdfs/bin/hadoop: No such file or directory

* hadoop fs -put /path/to/src_file hdfs://hdfs/ 
* hadoop fs -ls hdfs://hdfs/  should now list src_file

### Teardown Instructions
HDFS is *extremely* resilient, taking down the framework takes several steps. 

* In Marathon (or your other long-running process monitor) stop the hdfs scheduler application
* Teardown the hdfs framework in Mesos:  
	* `$ echo "<framework-id>" | curl -d@- -X POST http://<current mesos-master ip>:5050/master/teardown`
	* `$ echo "frameworkId=e7f829e3-4d1b-44c3-bd47-a1bf8df02c05-0001" | curl -d@- -X POST http://192.168.1.101:5050/master/teardown`
* Access your zookeeper command line:  
	* `$ /usr/lib/zookeeper/bin/zkCli.sh` 
* Remove hdfs-mesos framework state from zookeeper:  
	* `[zk: localhost:2181(CONNECTED) 0] rmr /hdfs-mesos`
	* `[zk: localhost:2181(CONNECTED) 1] quit` 
* Clear your data directories as specified in your  mesos-site.xml. 
	* This is necessary to relaunch HDFS in the same directory.
	* `/var/lib/hdfs/data`
	* `/opt/mesosphere`

### Further Reading
* [Stackoverflow Teardown Notes](http://stackoverflow.com/questions/25979560/kill-a-framework-in-mesos)
* [Github Mesos/HDFS](https://github.com/mesosphere/hdfs)
* [HDFS Design](https://hadoop.apache.org/docs/r1.2.1/hdfs_design.html)
* [HDFS User Guide](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HdfsUserGuide.html)
* [Apache Myriad](http://myriad.incubator.apache.org/)
* [Apache YARN](https://hadoop.apache.org/docs/r2.7.1/hadoop-yarn/hadoop-yarn-site/YARN.html)

## Install Cassandra
* Add Datastax repo:
* Example [etc/yum.repos.d/datastax.repo](/etc/yum.repos.d/datastax.repo)

			sudo tee /etc/yum.repos.d/datastax.repo <<-EOF
			[datastax] 
			name = DataStax Repo for Apache Cassandra
			baseurl = http://rpm.datastax.com/community
			enabled = 1
			gpgcheck = 0
			EOF

* `$ yum install cassandra`

### Further Reading
* TBD

## Install Kafka
* TBD

## References
* [Mesosphere Advance Course](https://open.mesosphere.com/advanced-course/)
* [Mesosphere Install](https://open.mesosphere.com/getting-started/install/)
* [Digital Ocean Notes](https://www.digitalocean.com/community/tutorials/how-to-configure-a-production-ready-mesosphere-cluster-on-ubuntu-14-04)
* [Mesosphere Basics](https://mesosphere.github.io/marathon/docs/application-basics.html)


