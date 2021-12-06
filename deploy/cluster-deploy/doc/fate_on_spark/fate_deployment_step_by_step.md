# FATE ON Spark/CDN FATE Deployment Guide

## 1. Basic Environment Configuration

### 1.1 hostname configuration (optional)

**1) Modify the hostname**

**Execute under 192.168.0.1 root user:**

```bash
hostnamectl set-hostname VM-0-1-centos
```

**Execute under 192.168.0.2 root user:**

```bash
hostnamectl set-hostname VM-0-2-centos
```

**2) Add the host mapping**

**Execute under the target server (192.168.0.1 192.168.0.2) root user:**

```bash
vim /etc/hosts
192.168.0.1 VM-0-1-centos
192.168.0.2 VM-0-2-centos
```

### 1.2 Shut down SELinux (optional)


**Execute under the root user of the target server (192.168.0.1 192.168.0.2):**

Confirm if SELinux is installed

CentOS system executes.

```bash
rpm -qa | grep selinux
```

For Ubuntu systems, run

```bash
apt list --installed | grep selinux
```

If SELinux is already installed, do the following.

```bash
setenforce 0
```

### 1.3 Modifying Linux system parameters

**Execute under the root user on the target server (192.168.0.1 192.168.0.2 192.168.0.3):**

```bash
vim /etc/security/limits.conf
* soft nofile 65536
* hard nofile 65536
```

```bash
vim /etc/security/limits.d/20-nproc.conf
* soft nproc unlimited
```

### 1.4 Turn off the firewall (optional)


** Execute under the root user of the target server (192.168.0.1 192.168.0.2 192.168.0.3) **

In case of CentOS system.

```bash
systemctl disable firewalld.service
systemctl stop firewalld.service
systemctl status firewalld.service
```

If it is an Ubuntu system.

```bash
ufw disable
ufw status
```

### 1.5 Software environment initialization

**Execute under the root user of the target server (192.168.0.1 192.168.0.2 192.168.0.3)**

**1) Create user**

```bash
groupadd -g 6000 apps
useradd -s /bin/bash -g apps -d /home/app app
passwd app
```

**2) Create a directory**

```bash
mkdir -p /data/projects/fate
mkdir -p /data/projects/install
chown -R app:apps /data/projects
```

**3) Install dependencies**

```bash
#centos
yum -y install gcc gcc-c++ make openssl-devel gmp-devel mpfr-devel libmpc-devel libaio numactl autoconf automake libtool libffi-devel snappy snappy -devel zlib zlib-devel bzip2 bzip2-devel lz4-devel libasan lsof sysstat telnet psmisc
#ubuntu
apt-get install -y gcc g++ make openssl supervisor libgmp-dev libmpfr-dev libmpc-dev libaio1 libaio-dev numactl autoconf automake libtool libffi- dev libssl1.0.0 libssl-dev liblz4-1 liblz4-dev liblz4-1-dbg liblz4-tool zlib1g zlib1g-dbg zlib1g-dev
cd /usr/lib/x86_64-linux-gnu
if [ ! -f "libssl.so.10" ];then
   ln -s libssl.so.1.0.0 libssl.so.10
   ln -s libcrypto.so.1.0.0 libcrypto.so.10
fi
```

## 2. Deploy dependent components

Note: The default installation directory for this guide is /data/projects/install, and the execution user is app, modify it according to the specific situation when installing.

### 2.1 Get the installation package


Execute on the target server (192.168.0.1 with extranet environment) under the app user:

```bash
mkdir -p /data/projects/install
cd /data/projects/install
wget https://webank-ai-1251170195.cos.ap-guangzhou.myqcloud.com/python-env-miniconda3-4.5.4.tar.gz
wget https://webank-ai-1251170195.cos.ap-guangzhou.myqcloud.com/jdk-8u192-linux-x64.tar.gz
wget https://webank-ai-1251170195.cos.ap-guangzhou.myqcloud.com/mysql-fate-8.0.13.tar.gz
wget https://webank-ai-1251170195.cos.ap-guangzhou.myqcloud.com/openresty-1.17.8.2.tar.gz
wget https://webank-ai-1251170195.cos.ap-guangzhou.myqcloud.com/pip-packages-fate-${version}.tar.gz
wget https://webank-ai-1251170195.cos.ap-guangzhou.myqcloud.com/FATE_install_${version}_release.tar.gz

#transfer to 192.168.0.1 and 192.168.0.2
scp *.tar.gz app@192.168.0.1:/data/projects/install
scp *.tar.gz app@192.168.0.2:/data/projects/install
```
Note: The current document needs to be deployed with FATE version>=1.7.0
### 2.2 Operating system parameter checking

**execute under the target server (192.168.0.1 192.168.0.2 192.168.0.3) app user**

```bash
# of file handles, not less than 65535, if not meet the need to refer to section 4.3 reset
ulimit -n
65535

#Number of user processes, not less than 64000, if not, you need to refer to section 4.3 to reset
ulimit -u
65535
```

### 2.3 Deploying MySQL

**Execute under the target server (192.168.0.1 192.168.0.2) app user**

**1) MySQL installation:**

```bash
# Create mysql root directory
mkdir -p /data/projects/fate/common/mysql
mkdir -p /data/projects/fate/data/mysql

#Unpack the package
cd /data/projects/install
tar xzvf mysql-*.tar.gz
cd mysql
tar xf mysql-8.0.13.tar.gz -C /data/projects/fate/common/mysql

#Configuration settings
mkdir -p /data/projects/fate/common/mysql/mysql-8.0.13/{conf,run,logs}
cp service.sh /data/projects/fate/common/mysql/mysql-8.0.13/
cp my.cnf /data/projects/fate/common/mysql/mysql-8.0.13/conf

#initialize
cd /data/projects/fate/common/mysql/mysql-8.0.13/
./bin/mysqld --initialize --user=app --basedir=/data/projects/fate/common/mysql/mysql-8.0.13 --datadir=/data/projects/fate/data/mysql > logs/init.log 2>&1
cat logs/init.log |grep root@localhost
#Note that the output message after root@localhost: is the initial password of the mysql user root, which needs to be recorded and used to change the password later

#Start the service
cd /data/projects/fate/common/mysql/mysql-8.0.13/
nohup ./bin/mysqld_safe --defaults-file=./conf/my.cnf --user=app >>logs/mysqld.log 2>&1 &

#change mysql root user password
cd /data/projects/fate/common/mysql/mysql-8.0.13/
./bin/mysqladmin -h 127.0.0.1 -P 3306 -S ./run/mysql.sock -u root -p password "fate_dev"
Enter Password: [Enter root initial password]

#Verify login
cd /data/projects/fate/common/mysql/mysql-8.0.13/
./bin/mysql -u root -p -S ./run/mysql.sock
Enter Password: [Enter root modified password:fate_dev]
```

**2) Build authorization and business configuration**

```bash
cd /data/projects/fate/common/mysql/mysql-8.0.13/
./bin/mysql -u root -p -S ./run/mysql.sock
Enter Password:[fate_dev]

# Create the fate_flow library
mysql>CREATE DATABASE IF NOT EXISTS fate_flow;

#Create remote user and authorization
1) 192.168.0.1 execute
mysql>CREATE USER 'fate'@'192.168.0.1' IDENTIFIED BY 'fate_dev';
mysql>GRANT ALL ON *. * TO 'fate'@'192.168.0.1';
mysql>flush privileges;

2) 192.168.0.2 execute
mysql>CREATE USER 'rate'@'192.168.0.2' IDENTIFIED BY 'rate_dev';
mysql>GRANT ALL ON *. * TO 'fate'@'192.168.0.2';
mysql>flush privileges;

# verify
mysql>select User,Host from mysql.user;
mysql>show databases;
mysql>use eggroll_meta;
mysql>show tables;
mysql>select * from server_node;

```

### 2.4 Deploying the JDK

**execute ** under the target server (192.168.0.1 192.168.0.2) app user:

```bash
#Create the jdk installation directory
mkdir -p /data/projects/fate/common/jdk
#Uncompress
cd /data/projects/install
tar xzf jdk-8u192-linux-x64.tar.gz -C /data/projects/fate/common/jdk
cd /data/projects/fate/common/jdk
mv jdk1.8.0_192 jdk-8u192
```

### 2.5 Deploying python

**execute ** under the target server (192.168.0.1 192.168.0.2) app user:

```bash
#Create the python virtualization installation directory
mkdir -p /data/projects/fate/common/python

#Install miniconda3
cd /data/projects/install
tar xvf python-env-*.tar.gz
cd python-env
sh miniconda3-4.5.4-Linux-x86_64.sh -b -p /data/projects/fate/common/miniconda3

#Install virtualenv and create a virtualized environment
/data/projects/fate/common/miniconda3/bin/pip install virtualenv-20.0.18-py2.py3-none-any.whl -f . --no-index

/data/projects/fate/common/miniconda3/bin/virtualenv -p /data/projects/fate/common/miniconda3/bin/python3.6 --no-wheel --no-setuptools --no-download /data/projects/fate/common/python/venv

#Install the dependency packages
cd /data/projects/install
tar xvf pip-packages-fate-*.tar.gz
source /data/projects/fate/common/python/venv/bin/activate
pip install python-env/setuptools-42.0.2-py2.py3-none-any.whl
pip install -r pip-packages-fate-${version}/requirements.txt -f ./pip-packages-fate-${version} --no-index
pip list | wc -l
```

### 2.6 Deploying Nginx

**execute ** under the target server (192.168.0.1 192.168.0.2) app user:

```bash
cd /data/projects/install
tar xzf openresty-*.tar.gz
cd openresty-*
./configure --prefix=/data/projects/fate/proxy \
                   --with-luajit \
                   --with-http_ssl_module \
                     --with-http_v2_module \
                     --with-stream \
                     --with-stream_ssl_module \
                     -j12
make && make install
```

### 2.7 Deploying RabbitMQ (either with pulsar)

See the deployment guide: [RabbitMQ_deployment_guide_zh](rabbitmq_deployment_guide_zh.md)

### 2.8 Deploying Pulsar (either with rabbitmq)

See the deployment guide: [Pulsar deployment](pulsar_deployment_guide_zh.md)

## 3 Deploying FATE

### 3.1 Software deployment

```
## Deploy the software
## On the target server (192.168.0.1 192.168.0.2) under the app user execute:
cd /data/projects/install
tar xf FATE_install_*.tar.gz
cd FATE_install_*
cp -r bin /data/projects/fate/
cp -r conf /data/projects/fate/
cp fate.env /data/projects/fate/
tar xvf python.tar.gz -C /data/projects/fate/
tar xvf fateboard.tar.gz -C /data/projects/fate
tar xvf proxy.tar.gz -C /data/projects/fate

# Set the environment variable file
#Execute on the target server (192.168.0.1 192.168.0.2) under the app user:
cat >/data/projects/fate/bin/init_env.sh <<EOF
fate_project_base=/data/projects/fate
export FATE_PROJECT_BASE=$fate_project_base
export FATE_DEPLOY_BASE=$fate_project_base

export PYTHONPATH=/data/projects/fate/fateflow/python:/data/projects/fate/eggroll/python:/data/projects/fate/fate/python
export EGGROLL_HOME=/data/projects/fate/eggroll
export EGGROLL_LOG_LEVEL=INFO
venv=/data/projects/fate/common/python/venv
export JAVA_HOME=/data/projects/fate/common/jdk/jdk-8u192
export PATH=$PATH:$JAVA_HOME/bin
source ${venv}/bin/activate
EOF
```

### 3.2 Nginx configuration file modifications
#### 3.2.1 Nginx base configuration file modifications
Configuration file: /data/projects/fate/proxy/nginx/conf/nginx.conf
This configuration file is used by Nginx to configure the service base settings and lua code, and generally does not need to be modified.
If you want to modify it, you can manually modify it by referring to the default nginx.conf, and use the command detect after the modification is done
```
/data/projects/fate/proxy/nginx/sbin/nginx -t
```

#### 3.2.2 Nginx routing configuration file modifications

Configuration file: /data/projects/fate/proxy/nginx/conf/route_table.yaml
This configuration file is used by NginX to configure routing information, either manually by referring to the following example, or by using the following command.

```
# Modify the execution under the target server (192.168.0.1) app user
cat > /data/projects/fate/proxy/nginx/conf/route_table.yaml << EOF
default:
  proxy:
    - host: 192.168.0.2
      port: 9390
10000:
  proxy:
    - host: 192.168.0.1
      port: 9390
  fateflow:
    - host: 192.168.0.1
      port: 9360
9999:
  proxy:
    - host: 192.168.0.2
      port: 9390
  fateflow:
    - host: 192.168.0.2
      port: 9360
EOF

# Modify the execution under the app user of the target server (192.168.0.2)
cat > /data/projects/fate/proxy/nginx/conf/route_table.yaml << EOF
default:
  proxy:
    - host: 192.168.0.1
      port: 9390
10000:
  proxy:
    - host: 192.168.0.1
      port: 9390
  fateflow:
    - host: 192.168.0.1
      port: 9360
9999:
  proxy:
    - host: 192.168.0.2
      port: 9390
  fateflow:
    - host: 192.168.0.2
      port: 9360
EOF
```

### 3.3 FATE-Board configuration file modification

1) conf/application.properties

- Service port

  server.port --- default

- access url of fateflow

  fateflow.url, host: http://192.168.0.1:9380, guest: http://192.168.0.2:9380

- Database connection string, account and password

  fateboard.datasource.jdbc-url, host: mysql://192.168.0.1:3306, guest: mysql://192.168.0.2:3306

  fateboard.datasource.username: fateboard

  fateboard.datasource.password：fate_dev
  
  The above parameter adjustment can be done manually by referring to the following example, or by using the following command
  
  Configuration file: /data/projects/fate/fateboard/conf/application.properties

```
# Modify the execution under the app user of the target server (192.168.0.1)
cat > /data/projects/fate/fateboard/conf/application.properties <<EOF
server.port=8080
fateflow.url=http://192.168.0.1:9380
spring.datasource.driver-Class-Name=com.mysql.cj.jdbc.
spring.http.encoding.charset=UTF-8
spring.http.encoding.enabled=true
server.tomcat.uri-encoding=UTF-8
fateboard.datasource.jdbc-url=jdbc:mysql://192.168.0.1:3306/fate_flow?characterEncoding=utf8&characterSetResults=utf8& autoReconnect=true&failOverReadOnly=false&serverTimezone=GMT%2B8
fateboard.datasource.username=fate
fateboard.datasource.password=fate_dev
server.tomcat.max-threads=1000
server.tomcat.max-connections=20000
EOF

# Modify the execution under the target server (192.168.0.2) app user
cat > /data/projects/fate/fateboard/conf/application.properties <<EOF
server.port=8080
fateflow.url=http://192.168.0.2:9380
spring.datasource.driver-Class-Name=com.mysql.cj.jdbc.
spring.http.encoding.charset=UTF-8
spring.http.encoding.enabled=true
server.tomcat.uri-encoding=UTF-8
fateboard.datasource.jdbc-url=jdbc:mysql://192.168.0.2:3306/fate_flow?characterEncoding=utf8&characterSetResults=utf8& autoReconnect=true&failOverReadOnly=false&serverTimezone=GMT%2B8
fateboard.datasource.username=fate
fateboard.datasource.password=fate_dev
server.tomcat.max-threads=1000
server.tomcat.max-connections=20000
EOF
```

2) service.sh

```
#Modify the execution under the target server (192.168.0.1 192.168.0.2) app user
cd /data/projects/fate/fateboard
vi service.sh
export JAVA_HOME=/data/projects/fate/common/jdk/jdk-8u192
```

## 4. FATE configuration file modification
 
  Configuration file: /data/projects/fate/conf/service_conf.yaml
  
##### running configuration
- FATE engine related configuration.

```yaml
default_engines:
  compute: spark
  federation: rabbitmq #(or pulsar)
  storage: HDFS
```

- Listening ip, port of FATE-Flow

- Listening ip, port of FATE-Board

- db's connection ip, port, account and password

##### Dependent service configuration

**conf/service_conf.yaml**
```yaml
fate_on_spark:
  spark:
    home:
    cores_per_node: 40
    nodes: 1
  hdfs:
    name_node: hdfs://fate-cluster
    path_prefix:
  # rabbitmq and pulsar
  rabbitmq:
    host: 127.0.0.1
    mng_port: 12345
    port. 5672
    user: fate
    password: fate
    route_table:
  pulsar:
    host: 127.0.0.1
    port. 6650
    mng_port: 8080
    cluster: standalone
    tenant: fl-tenant
    topic_ttl: 5
    route_table:     
```
- Spark related configuration
    - Home is the absolute path to Spark's home page
    - cores_per_node is the number of cpu cores in each node of Spark cluster
    - node is the number of nodes in Spark cluster

- HDFS-related configuration
    - name_node is the full address of the hdfs namenode
    - path_prefix is the default storage path prefix, if not configured then the default is /.

- RabbitMQ related configuration
    - host: host_ip
    - management_port: 
    - mng_port: Management port
    - port. Service port
    - user: admin user
    - password: administrator password
    - route_table: Routing table information, default is empty
    
- pulsar-related configuration
    - host: host ip
    - port: brokerServicePort
    - mng_port: webServicePort
    - cluster: cluster or standalone
    - tenant: partner needs to use the same tenant
    - topic_ttl: recycling resource parameter
    - route_table: Routing table information, default is empty.
    
**conf/rabbitmq_route_table.yaml**
```yaml
10000:
  Host: 127.0.0.1
  port. 5672
9999:
  Host: 127.0.0.2
  port. 5672
```

**conf/pulsar_route_table.yaml**
```yaml
9999:
  # host can be a domain name, e.g. 9999.fate.org
  host: 192.168.0.4
  port. 6650
  sslPort: 6651
  # Set proxy address for this pulsar cluster
  proxy.""

10000:
  # The host can be a domain name, such as 10000.fate.org
  host: 192.168.0.3
  port: 6650
  sslPort: 6651
  proxy.""

default.
  # Compose hosts and proxies for parties that do not exist in the routing table
  # In this example, the host for the 8888 party will be 8888.fate.org
  proxy." proxy.fate.org:443"
  Domain." fate.org"
  port. 6650
  sslPort: 6651
```



- Proxy related configuration (ip and port)

**conf/service_conf.yaml**
```yaml
fateflow.
  proxy: nginx
fate_on_spark:
  nginx: 
    Host: 127.0.0.1
    port: 9390
```

##### spark dependency distribution model
- "conf/service_conf.yaml"
```yaml
dependent_distribution: true # Recommended to use true
```

**Note:If this configuration is "true", the following actions can be ignored**.

- Dependency preparation:copy the entire fate directory to each working node, keeping the directory structure consistent

- spark configuration modification: spark/conf/spark-env.sh
```shell script
export PYSPARK_PYTHON=/data/projects/fate/common/python/venv/bin/python
export PYSPARK_DRIVER_PYTHON=/data/projects/fate/common/python/venv/bin/python
```


## 5. Start the service

execute under the target server (192.168.0.1 192.168.0.2) app user

```
## Start FATE service, FATE-Flow depends on MySQL to start
Source code /data/projects/fate/bin/init_env.sh
cd /data/projects/fate/fateflow/bin
sh service.sh start
cd /data/projects/fate/fateboard
sh service.sh start
cd /data/projects/fate/proxy
./nginx/sbin/nginx -c /data/projects/fate/proxy/nginx/conf/nginx.conf
```

## 6. Problem location

1) FATE-Flow log

/data/projects/fate/fateflow/logs

2) FATE-Board logs

/data/projects/fate/fateboard/logs

3) NginX logs

/data/projects/fate/proxy/nginx/logs

## 7. Testing

### 7.1 Toy_example deployment verification

Reference document:[toy test](../fate_on_eggroll/Fate-allinone_deployment_guide_install.md#61-verify-toy_example-deployment)


### 7.2. FateBoard testing

Fate-Voard is a web service. If the fateboard service is successfully started, you can view the task information by visiting http://192.168.0.1:8080 and http://192.168.0.2:8080, if there is a firewall in place. If fateboard and fateflow are not deployed on the same server, you need to set the login information of the host where fateflow is deployed on the fateboard page: the gear button on the top right side of the page --> add --> fill in the fateflow host ip, os user, ssh port, password.


## 8. System Operation and Maintenance
================

### 8.1 Service Management

execute under the target server (192.168.0.1 192.168.0.2) app user

#### 8.1.1 FATE Service Management

1) Start/shutdown/view/restart the fate_flow service

```bash
source /data/projects/fate/init_env.sh
cd /data/projects/fate/fateflow/bin
sh service.sh start|stop|status|restart
```

If you start module by module, you need to start eggroll first and then fateflow. fateflow depends on eggroll to start.

2) Start/shutdown/restart FATE-Board service

```bash
cd /data/projects/fate/fateboard
sh service.sh start|stop|status|restart
```

3) Start/shutdown/restart the NginX service

```
cd /data/projects/fate/proxy
./nginx/sbin/nginx -s reload
./nginx/sbin/nginx -s stop
```

#### 8.1.2 MySQL Service Management

Start/shutdown/view/restart MySQL services

```bash
cd /data/projects/fate/common/mysql/mysql-8.0.13
sh ./service.sh start|stop|status|restart
```

### 8.2 Viewing processes and ports

execute under the target server (192.168.0.1 192.168.0.2) app user

##### 8.2.1 View processes

```
# Check if the process is started according to the deployment plan
ps -ef | grep -i fate_flow_server.py
ps -ef | grep -i fateboard
ps -ef | grep -i nginx
```

### 8.2.2 Viewing process ports

```
### 8.2.2 Checking the process ports according to the deployment plan
### 8.2.2 Checking process ports
netstat -tlnp | grep 9360
#fateboard
netstat -tlnp | grep 8080
#nginx
netstat -tlnp | grep 9390
```


### 8.3 Service logs

| service | logpath |
| ------------------ | -------------------------------------------------- |
| fate_flow&task_logs | /data/projects/fate/fateflow/logs |
| fateboard | /data/projects/fate/fateboard/logs |
| nginx | /data/projects/fate/proxy/nginx/logs | mysql | /data/projects/fate/proxy/nginx/logs
| mysql | /data/projects/fate/common/mysql/mysql-8.0.13/logs |

## 9. Appendices

### 9.1 Packaged builds

See the [build guide](./build.md) 