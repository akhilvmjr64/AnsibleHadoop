### Task: 11 ARTH
# Configuring Hadoop with Ansible
## Ansible playbook to configure hadoop on the managed nodes
### 🔰 11.1 Configure Hadoop and start cluster services using Ansible Playbook

This is a task in which I configured the hadoop cluster using the ansible playbook. Here I am explainging how I did that in detail.

Firstly to create a hadoop cluster we need to install hadoop as hadoop requires java we need to install java and hadoop in the managed nodes either whether it maybe a data node or the name node. Install a software we need to download the software. As I was using a RedHat OS hence I downloaded the rpm packages available in the internet using the curl command, then I installed the dowloaded rpm using the rpm command in redhat. Following is the part of playbook which I used to do this:

```yaml
- hosts : all
  tasks :
          - name : "Installing hadoop downloading the required packages"
            shell :
                    cmd : "curl -o /tmp/hadoop.rpm https://archive.apache.org/dist/hadoop/core/hadoop-1.2.1/hadoop-1.2.1-1.i386.rpm; curl -o /tmp/java-jdk.rpm http://anfadmin.ucsd.edu/linux/RHEL/7/x86_64/jdk-8u171-linux-x64.rpm"
            ignore_errors: yes
          - name : "Installing java jdk dependency for hadoop"
            shell :
                  cmd : "rpm -ivh /tmp/java-jdk.rpm"
            ignore_errors: yes
          - name : "Installing hadoop"
            shell :
                  cmd : "rpm -ivh /hadoop_packages/packages/ha* --force; true"
            ignore_errors: yes
```

Also I have also included to stop the namenode and datanode if the they are running to the above part in the playbook which is as follows

```yaml
          - name : "Stopping the name node or data node if it already exists"
            shell :
                    cmd : "hadoop-daemon.sh stop namenode; hadoop-daemon.sh stop datanode"
```
Then, I declared some host groups in the inventory file of the ansible where for the sake of NameNode I have listed a host and for DataNode I have listed someother hosts. Then I configured the nodes with their respective configurations using the playbooks as follows

## Configuring the NameNode :

I have configured the name node by firstly creating the directory in the name node using the file module then I copied the configuration files `hdfs-site.xml` and `core-site.xml` to the namenodes with the required values in them required for the nam node I have created templates for both of them which are named as `hadoop_hdfs_name.xml` and `hadoop_core_name.xml` . After configuring the namenode I have formatted the name node using the command `echo 'Y' | hadoop namenode -format` and started the name node using the command `hadoop-daemon.sh start namenode` . After starting the namenode I stopped the firewalld service and I have set the enforcing to permissive mode. All this is done using the following part of the playbook

```yaml
- hosts : hadoop_name_node
  tasks :
          - debug :
                  msg : "Configuring the name node"
          - name : "Creating the name node directory"
            shell :
                    cmd : "rm -rf /name_dir"
          - file :
                  path : "/name_dir"
                  state : "directory"
          - name : "Configuring the namenode changing hdfs-site.xml"
            template :
                    src : "./hadoop_hdfs_name.xml"
                    dest : "/etc/hadoop/hdfs-site.xml"
          - name : "Configuring the namenode changing core-site.xml"
            template :
                    src : "./hadoop_core_name.xml"
                    dest : "/etc/hadoop/core-site.xml"
          - name : "Formatting namenode"
            shell :
                    cmd : "echo 'Y' | hadoop namenode -format"
          - name : "Starting namenode"
            shell :
                    cmd : "hadoop-daemon.sh start namenode"
          - name : "Stopping firewall"
            shell :
                    cmd : "systemctl stop firewalld"
          - name : "Setting SELinux to permissive"
            shell :
                    cmd : "setenforce 0"
```

The xml files used for configuring the namenode are as follows

`hdfs-site.xml` -> `hadoop_hdfs_name.xml`

```xml
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<!-- Put site-specific property overrides in this file. -->
<configuration>
<property>
<name>dfs.name.dir</name>
<value>/name_dir</value>
</property>
</configuration>
```

`core-site.xml` -> `hadoop_core_name.xml`

```xml
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<!-- Put site-specific property overrides in this file. -->
<configuration>
<property>
<name>fs.default.name</name>
<value>{{ansible_facts['enp0s3']['ipv4']['address']}}:9001</value>
</property>
</configuration>
```

Similarly I have configured the datanodes

## Configuring the datanodes :

I have configured the data node by firstly creating the directory in the name node using the file module then I copied the configuration files `hdfs-site.xml` and `core-site.xml` to the datanodes with the required values in them required for the datanodes I have created templates for both of them which are named as `hadoop_hdfs_data.xml` and `hadoop_core_data.xml` . After configuring the datanode I have started the datanode using the command `hadoop-daemon.sh start datanode` . After starting the datanode I stopped the firewalld service and I have set the enforcing to permissive mode. All this is done using the following part of the playbook
```yaml
- hosts : hadoop_data_node
  vars :
          name_ip : "{{ groups['hadoop_name_node'][0] }}"
  tasks :
          - debug :
                  msg : "Configuring the data node"
          - name : "Creating data node directory in each data node"
            shell :
                    cmd : "rm -rf /data_dir"
          - file :
                  path : "/data_dir"
                  state : "directory"
          - name : "Configuring the datanode changing hdfs-site.xml"
            template :
                    src : "./hadoop_hdfs_data.xml"
                    dest : "/etc/hadoop/hdfs-site.xml"
          - name : "Configuring the datanode changing core-site.xml"
            template :
                    src : "./hadoop_core_data.xml"
                    dest : "/etc/hadoop/core-site.xml"
          - name : "Starting datanode"
            shell :
                    cmd : "hadoop-daemon.sh start datanode"
          - name : "Stopping firewall"
            shell :
                    cmd : "systemctl stop firewalld"
          - name : "Setting SELinux to permissive"
            shell :
                    cmd : "setenforce 0"
```

I have used a variable to store the namenodes IP address and used that variable in the configuration files of the datanodes

The xml files used for configuring the datanodes are as follows

`hdfs-site.xml` -> `hadoop_hdfs_data.xml`
```xml
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<!-- Put site-specific property overrides in this file. -->
<configuration>
<property>
<name>dfs.data.dir</name>
<value>/data_dir</value>
</property>
</configuration>
```
`core-site.xml` -> `hadoop_core_data.xml`
```xml
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<!-- Put site-specific property overrides in this file. -->
<configuration>
<property>
<name>fs.default.name</name>
<value>{{name_ip}}:9001</value>
</property>
</configuration>
```
In this way I have configured the hadoop cluster
