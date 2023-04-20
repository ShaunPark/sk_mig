본 가이드에서는 Ansible을 통해 Confluent Platform을 설치 및 구성하는 방법을 다룹니다. 

- **Confluent Platform Version: 7.3.2**

---

---

# 1. Prerequisites

![Confluent Ansible 7.3 을 사용하는 경우의 Prerequisites ](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/33b76007-ab63-4f7d-a2bf-11d4ede82712/Untitled.png)

Confluent Ansible 7.3 을 사용하는 경우의 Prerequisites 

- **Ansible 2.11 + (Control Node)**
- **Python 3.6 +  (Control Node/Managed Nodes)**
: Confluent 7.2부터는 Python 2.x 지원하지 않으며, 본 가이드에서는 3.7.4 버전으로 진행.
- **Ansible Control Node와 Confluent Platform Nodes 간의 양방향 SSH 통신 가능**
    - **Public SSH Key 등록하기**
        - Control Node에서 key 생성:  `$ ssh-keygen -t rsa`
        - Control Node에서 Confluent Platform Node 로 Public key 전송:  
        `$ ssh-copy-id -i ~/.ssh/id_rsa.pub confluent@<zookeeper-node-host>, 
         $ ssh-copy-id -i ~/.ssh/id_rsa.pub confluent@<broker-node-host>`
            
            ```bash
            *# --- 확인* 
            
            ***@ Confluent Platform Node: ~/.ssh/authorized_keys***
            *ssh-rsa AAAAB3NzaC1yc2EAAAADAQAB~~~ confluent@control02
            
            **@ Ansible Control Node: ~/.ssh/known_hosts**
            zk01, 192.168.137.100 ecdsa-sha2-nistp256 AAAAE2VjZHNhLXN~~*
            ```
            
- **SSH USER의 sudo 권한**
- **JDK 11 이상**
: 본 매뉴얼에서는 JDK 17 설치하여 진행
- **[Optional] Confluent Platform Software (**packages.confluent.io)**에 대한 인터넷 연결**
: 폐쇄망인 경우**,** [https://docs.confluent.io/ansible/current/ansible-airgap.html#air-gapped-deployment-of-ansible](https://docs.confluent.io/ansible/current/ansible-airgap.html#air-gapped-deployment-of-ansible) 참조
- **Time Synchronization**

**[ 설치/구성 파일 디렉터리 구조 ]**

```bash
**ansible**   *# 인터넷 연결 환경* ****
├── ansible.cfg
├── collections
│   └── ansible_collections
│       └── confluent
│           └── platform
│               ├── hosts.yml
│               ├── docs
│               ├── galaxy_importer
│               ├── meta
│               ├── molecule
│               ├── playbooks
│               ├── plugins
│               ├── roles
│               ├── test_roles
│               └── tests
├── source 
└── venv
**private-ansible** *# 폐쇄망 환경* ****
├── ansible.cfg
├── collections
│   └── confluent
│       └── platform
│           ├── hosts.yml
│           ├── docs
│           ├── galaxy_importer
│           ├── meta
│           ├── molecule
│           ├── playbooks
│           ├── plugins
│           ├── roles
│           ├── test_roles
│           └── tests
├── packages 
├── requirements.txt
├── source 
└── venv
```

# 2. 환경 준비

## 2-1. Python

- ***python3 (Ansible Control Node 및  Managed Nodes)***

```
--- root 계정 

# **cd /usr/src
# wget https://www.python.org/ftp/python/3.7.4/Python-3.7.4.tgz
# tar xzf Python-3.7.4.tgz**

# **cd Python-3.7.4
# ./configure --enable-optimizations**
--- If error, **sudo yum -y groups install "Development Tools"**

--- 기존의 /usr/bin/python 보호하며 추가설치 진행
# **yum -y install libffi-devel zlib-devel openssl openssl-devel
# make altinstall** 
*...
Collecting setuptools
Collecting pip
Installing collected packages: setuptools, pip
Successfully installed pip-19.0.3 setuptools-40.8.0

#* **./python -V**
*Python 3.7.4

#* **ln -s /usr/src/Python-3.7.4/python /usr/bin/python3**

*#* **python3**
*Python 3.7.4 (default, Oct 28 2022, 09:41:39) 
...*
```

```
**# python 의 디폴트 버전을 3.7 로 변경하는 경우 yum 명령어 사용이 불가해지는 경우**

[root@zk01 ~]# python --version
Python 3.7.4

[root@zk01 ~]# yum -y install nginx
  File "/bin/yum", line 30
    except KeyboardInterrupt, e:
                            ^
SyntaxError: invalid syntax

**### 해결 : 아래 두 파일에서 이용하는 python 의 버전을 2버전으로 명시해준다. 

$** vi /usr/bin/yum 
**$** vi /usr/libexec/urlgrabber-ext-down

*#!/usr/bin/python
=>
#!/usr/bin/python2*
```

- ***pip3 (Ansible Control Node): 파이썬으로 작성된 패키지 라이브러리들을 관리해주는 시스템***

```
**# cd ~ 
# curl https://bootstrap.pypa.io/pip/get-pip.py -o get-pip.py
# python get-pip.py**

*Installing collected packages: wheel, pip
  Attempting uninstall: pip
    Found existing installation: pip 19.0.3
    Uninstalling pip-19.0.3:
      Successfully uninstalled pip-19.0.3
Successfully installed pip-22.3 wheel-0.37.1*
```

- ***virtualenv (Ansible Control Node)***

```bash
# --- regular user로 변경하여 진행: 본 가이드에서는 confluent 계정으로 진행 

**$ cd ~**
**$ pip3 install virtualenv**
*Successfully installed distlib-0.3.6 filelock-3.8.0 importlib-metadata-5.0.0 platformdirs-2.5.2 typing-extensions-4.4.0 virtualenv-20.16.6 zipp-3.10.0*
```

- ***venv (Ansible Control Node)*** **: 프로젝트 별로 파이썬 패키지 관리용**

```bash
# --- ansible용 디렉토리 생성 

**$ mkdir ~/ansible ; cd ~/ansible** 

**$ virtualenv --python /usr/bin/python3 venv**
*created virtual environment CPython3.6.8.final.0-64 in 803ms
  creator CPython3Posix(dest=/home/confluent/ansible/venv, clear=False, no_vcs_ignore=False, global=False)
  seeder FromAppData(download=False, pip=bundle, setuptools=bundle, wheel=bundle, via=copy, app_data_dir=/home/confluent/.local/share/virtualenv)
    added seed packages: pip==21.3.1, setuptools==59.6.0, wheel==0.37.1
  activators BashActivator,CShellActivator,FishActivator,NushellActivator,PowerShellActivator,PythonActivator*

**$ ll**
*total 0
drwxrwxr-x. 4 confluent confluent 64 Oct 28 10:06 venv*
```

## 2-2. Ansible 설치

- **venv를 통한 가상 환경 활성화**

```bash
**$ cd ~/ansible**

**$ source venv/bin/activate**

**(venv) [confluent@control ansible]$ pip3 install ansible-core==2.11.0**

*Successfully installed MarkupSafe-2.1.1 PyYAML-6.0 ansible-core-2.11.0 cffi-1.15.1 cryptography-38.0.1 jinja2-3.1.2 packaging-21.3 pycparser-2.21 pyparsing-3.0.9 resolvelib-0.5.4

# ---- ansible.cfg 파일 생성:*

**(venv) [confluent@control ansible]$ *vi ansible.cfg***
*[defaults]
hash_behaviour = merge
collections_paths = /home/confluent/ansible/collections  # <--- ansible-collection 파일들이 설치 될 위치 지정
host_key_checking = False 
deprecation_warnings = False
stdout_callback = debug # <--- 앤서블 에러 출력화면 설정*
```

<aside>
⚠️ **venv 가상 환경 비활성화하는 방법** :  *$ deactivate*

(venv) [confluent@control ansible]$ **deactivate**
[confluent@control ansible]$

</aside>

## 2-3. Ansible Playbook 설치

```bash
**(venv) [confluent@control ansible]$ ansible-galaxy collection install confluent.platform:7.3.2**

****(venv) [confluent@control ansible]$ cd /home/confluent/ansible/ansible-galaxy/ansible_collections/confluent/platform**

****(venv) [confluent@control platform]$ ll**
*total 160
-rw-r--r--.  1 confluent confluent    189 Oct 28 10:16 ansible.cfg
drwxr-xr-x.  3 confluent confluent   4096 Oct 28 10:16 docs
-rw-r--r--.  1 confluent confluent 121562 Oct 28 10:16 FILES.json
drwxr-xr-x.  2 confluent confluent     33 Oct 28 10:16 galaxy_importer
-rw-r--r--.  1 confluent confluent   5189 Oct 28 10:16 Jenkinsfile
-rw-r--r--.  1 confluent confluent  11357 Oct 28 10:16 LICENSE.md
-rw-r--r--.  1 confluent confluent    874 Oct 28 10:16 MANIFEST.json
drwxr-xr-x.  2 confluent confluent     25 Oct 28 10:16 meta
drwxr-xr-x. 57 confluent confluent   4096 Oct 28 10:16 molecule
drwxr-xr-x.  3 confluent confluent    277 Oct 28 10:16 playbooks
drwxr-xr-x.  4 confluent confluent     35 Oct 28 10:16 plugins
-rw-r--r--.  1 confluent confluent   1083 Oct 28 10:16 README.md
drwxr-xr-x. 14 confluent confluent    229 Oct 28 10:16 roles
drwxr-xr-x.  5 confluent confluent     86 Oct 28 10:16 test_roles
drwxr-xr-x.  3 confluent confluent     20 Oct 28 10:16 tests*
```

- test-hosts.yml작성한 다음, ping 테스트 진행

```bash
**(venv) [confluent@control ansible]$ cd /home/confluent/ansible/ansible-galaxy/ansible_collections/confluent/platform
(venv) [confluent@control platform]$ vi test-ping.yml**

*zookeeper:
  hosts:
    zk01:

kafka_broker:
  hosts:
    br01: 
      broker_id: 1

schema_registry:
  hosts:
    sr-ksql01:

kafka_connect:
  hosts:
    cn01: 

control_center:
  hosts:
    control01:*

**(venv) [confluent@red platform]$ ansible -i test-ping.yml all -m ping**
```

<aside>
⚠️ **Python 의 default version이 3인 Ansible Managed Nodes 대상으로 yum 모듈이 들어간 task 실행하는 경우 아래 ERROR 발생:** 

`The Python 2 bindings for rpm are needed for this module. If you require Python 3 support use the `dnf` Ansible module instead.. The Python 2 yum module is needed for this module. If you require Python 3 support use the `dnf` Ansible module instead."`

🔎 **Cause: yum module은 python 2에서만 정상 동작**

****💡 **Solution (1) : 각 Ansible Managed Nodes 의 python default version을 2로 원복 후, ansible-playbook 실행 시 python 변수를 /usr/bin/python3 로 명시합니다.**

```
$ ansible-playbook -i hosts.yml confluent.platform.all --tags=zookeeper -e ansible_python_interpreter=/usr/bin/python3
```

💡 **Solution** **(2): 각 Ansible Managed Nodes 에 dnf 설치 후, Ansible Playbook 의 task에 선언된 yum 모듈을 모두 dnf 로 수정합니다.**

```
$ find -type f  -name "main.yml" | xargs sed -i "s/yum:/dnf:/g"
$ find -type f  -name "redhat.yml" | xargs sed -i "s/yum:/dnf:/g"

....
- name: Install OpenSSL and Unzip
  ~~yum: >>~~  dnf: 
    name:
      - openssl
      - unzip
  tags: package
....
```

</aside>

# 3. General Configuration

### 3-1. Confluent 설치 및 실행 방식 설정

- ansible 실행 및 confluent 플랫폼별 실행할 OS 계정 설정

```bash
all:
  vars:
    ### ansible
    ansible_connection: ssh
    ansible_user: confluent
    ansible_become: true # confluent 계정에서 sudo를 통해 root 사용자로 전환
    mask_secrets: false
    mask_sensitive_logs: false
    mask_sensitive_diff: false

    ### os user and group
    archive_owner: confluent
    archive_group: confluent
    zookeeper_user: confluent
    zookeeper_group: confluent
    kafka_broker_user: confluent
    kafka_broker_group: confluent
    schema_registry_user: confluent
    schema_registry_group: confluent
    kafka_connect_user: confluent
    kafka_connect_group: confluent
    ksql_user: confluent
    ksql_group: confluent
    control_center_user: confluent
    control_center_group: confluent
```

- java 설치 실행 여부 : 각 Ansible Managed Nodes에 설치되어있는 JDK17 경로를 명시합니다.

```bash
		### jdk
    custom_java_path: "/usr/lib/jvm/jdk-17-oracle-x64"
```

- confluent 설치 버전 및 방식 설정

```bash
		confluent_server_enabled: true
    confluent_package_version: "7.3.2"   

		confluent_cli_download_enabled: true
    confluent_cli_version: "2.30.1"
    confluent_cli_base_path: "/engn/confluent_cli"
    confluent_cli_path: "/usr/local/bin/confluent"

		# installation_method: yum/apt방식(package)이 아닌 아카이브 파일을 다운로드 받아 설치 
    installation_method: archive
    archive_destination_path: "/engn"
    confluent_archive_file_remote: true

    # deployment_strategy: 한 번에 한 호스트에서만 task가 실행되는 방식(rolling), 동시 배포 방식(parallel) 중 설정 
    deployment_strategy: rolling
```

### 3-2. Listener 설정

- Kafka Listeners 설정

```
		### kafka listeners
    kafka_broker_configure_multiple_listeners: true
    kafka_broker_configure_control_plane_listener: false
    kafka_broker_inter_broker_listener_name: broker
    kafka_broker_custom_listeners:
      internal:
        name: INTERNAL
        port: 9092
        sasl_protocol: none
      broker:
        name: BROKER
        port: 9093
        sasl_protocol: none

    ### kafka internal listener
    schema_registry_kafka_listener_name: internal
    kafka_connect_kafka_listener_name: internal
    kafka_rest_kafka_listener_name: internal
    ksql_kafka_listener_name: internal
    ksql_processing_log_kafka_listener_name: internal
    control_center_kafka_listener_name: internal
```

### 3-3. 모니터링 및 로그 관련 파일 관련 설정

- jmxexporter 관련 설정

```bash
		jmxexporter_version: 0.17.2 
    jmxexporter_enabled: true
    jmxexporter_url_remote: true # 인터넷에서 다운받도록 설정 
    jmxexporter_jar_path: "{{archive_destination_path}}/prometheus/jmx_prometheus_javaagent.jar"
    zookeeper_jmxexporter_config_path: "{{archive_destination_path}}/prometheus/zookeeper.yml"
    zookeeper_jmxexporter_port: 1234
		...
```

- log4j 파일 관련 설정
: source_path 로 지정한 경로 아래에 배포할 파일들을 위치시킵니다.

```bash
		zookeeper_copy_files:
      - source_path: "/home/confluent/ansible/source/log4j/zookeeper-log4j.properties"
        destination_path: "{{archive_destination_path}}/log4j/zookeeper-log4j.properties"
    kafka_broker_copy_files:
      - source_path: "/home/confluent/ansible/source/log4j/kafka-log4j.properties"
        destination_path: "{{archive_destination_path}}/log4j/kafka-log4j.properties"
		...
```

<aside>
<img src="/icons/arrow-down_lightgray.svg" alt="/icons/arrow-down_lightgray.svg" width="40px" /> **[DOWNLOAD] log4j.properties**

[zookeeper-log4j.properties](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/733fef1a-7bd4-41f4-96c5-bd12a9b51b9b/zookeeper-log4j.properties)

[kafka-log4j.properties](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/694b53a6-5bab-43e8-9627-bdb0eb23f0be/kafka-log4j.properties)

[kafka-connect-log4j.properties](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/90518b8d-650d-4b09-8a9e-2285049825e3/kafka-connect-log4j.properties)

[schema-registry-log4j.properties](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/5eeb9492-2266-4c22-86a3-7b46410349ad/schema-registry-log4j.properties)

[ksqldb-log4j.properties](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/f58e1c2b-5e54-4e3c-a9e2-6a63f9e1e782/ksqldb-log4j.properties)

[control-center-log4j.properties](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/2ed7a2e5-9be4-4435-875d-e915c13dd057/control-center-log4j.properties)

</aside>

### 3-4. Component 별 설정

- zookeeper 설정 정보

```
		zookeeper_log_dir: /logs/zookeeper
    zookeeper_chroot: /confluent

    zookeeper_service_environment_overrides:
      KAFKA_HEAP_OPTS: "-Xms512m -Xmx512m"
      KAFKA_JVM_PERFORMANCE_OPTS: "-server -XX:+UseG1GC -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35 -XX:+ExplicitGCInvokesConcurrent -XX:MaxInlineLevel=15 -Djava.awt.headless=true"
      KAFKA_GC_LOG_OPTS: "-Xlog:gc*:file={{zookeeper_log_dir}}/zookeeper-gc.log:time,tags:filecount=10,filesize=100M"
      KAFKA_JMX_OPTS: "-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false"
      KAFKA_LOG4J_OPTS: "-Dlog4j.configuration=file:{{archive_destination_path}}/log4j/zookeeper-log4j.properties"

		zookeeper_custom_properties:
      dataDir: /data/zookeeper
      tickTime: 2000
      initLimit: 5
      syncLimit: 2
      maxClientCnxns: 0
      autopurge.snapRetainCount: 10
      autopurge.purgeInterval: 1
      admin.enableServer: "false"
      4lw.commands.whitelist: ruok,stat,srvr
```

- broker 설정 정보

```
		kafka_broker_log_dir: /logs/kafka

    kafka_broker_service_environment_overrides:
      KAFKA_HEAP_OPTS: "-Xms4g -Xmx4g"
      KAFKA_JVM_PERFORMANCE_OPTS: "-server -XX:+UseG1GC -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35 -XX:+ExplicitGCInvokesConcurrent -XX:MaxInlineLevel=15 -Djava.awt.headless=true"
      KAFKA_GC_LOG_OPTS: "-Xlog:gc*:file={{kafka_broker_log_dir}}/kafka-gc.log:time,tags:filecount=10,filesize=100M"
      KAFKA_JMX_OPTS: "-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false"
      KAFKA_LOG4J_OPTS: "-Dlog4j.configuration=file:{{archive_destination_path}}/log4j/kafka-log4j.properties"

		kafka_broker_custom_properties:
      log.dirs: /data/kafka
      log.retention.hours: 168
      log.retention.check.interval.ms: 300000
      log.segment.bytes: 1073741824
      log.cleanup.policy: delete
      num.partitions: 3
      num.recovery.threads.per.data.dir: 2
      default.replication.factor: 3
      min.insync.replicas: 2
      replica.lag.time.max.ms: 30000
      unclean.leader.election.enable: "false"
      auto.create.topics.enable: "false"
      group.initial.rebalance.delay.ms: 3000
      offsets.topic.replication.factor: 3
      transaction.state.log.min.isr: 2
      transaction.state.log.replication.factor: 3
      confluent.balancer.enable: "true"
      confluent.balancer.heal.uneven.load.trigger: EMPTY_BROKER
      confluent.balancer.disk.max.load: 0.85
      confluent.balancer.topic.replication.factor: 3
      confluent.tier.enable: "false"
```

- schema-registry 설정 정보

```
		schema_registry_log_dir: /logs/schema-registry
    schema_registry_service_environment_overrides:
      SCHEMA_REGISTRY_HEAP_OPTS: "-Xms512m -Xmx512m"
      SCHEMA_REGISTRY_JVM_PERFORMANCE_OPTS: "-server -XX:+UseG1GC -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35 -XX:+ExplicitGCInvokesConcurrent -XX:MaxInlineLevel=15 -Djava.awt.headless=true"
      SCHEMA_REGISTRY_GC_LOG_OPTS: "-Xlog:gc*:file={{schema_registry_log_dir}}/schema-registry-gc.log:time,tags:filecount=10,filesize=100M"
      SCHEMA_REGISTRY_JMX_OPTS: "-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false"
      SCHEMA_REGISTRY_LOG4J_OPTS: "-Dlog4j.configuration=file:{{archive_destination_path}}/log4j/schema-registry-log4j.properties"

    schema_registry_custom_properties:
      schema.compatibility.level: full
```

- connect worker 설정 정보

```
		kafka_connect_log_dir: /logs/kafka-connect
    kafka_connect_service_environment_overrides:
      KAFKA_HEAP_OPTS: "-Xms4g -Xmx4g"
      KAFKA_JVM_PERFORMANCE_OPTS: "-server -XX:+UseG1GC -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35 -XX:+ExplicitGCInvokesConcurrent -XX:MaxInlineLevel=15 -Djava.awt.headless=true"
      KAFKA_GC_LOG_OPTS: "-Xlog:gc*:file={{kafka_connect_log_dir}}/connect-gc.log:time,tags:filecount=10,filesize=100M"
      KAFKA_JMX_OPTS: "-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false"
      KAFKA_LOG4J_OPTS: "-Dlog4j.configuration=file:{{archive_destination_path}}/log4j/kafka-connect-log4j.properties"
      CLASSPATH: "{{archive_destination_path}}/confluent-{{confluent_package_version}}/share/java/kafka-connect-replicator/*"

    kafka_connect_custom_properties:
      config.storage.replication.factor: 3
      offset.storage.partitions: 25
      offset.storage.replication.factor: 3
      offset.flush.interval.ms: 10000
      status.storage.partitions: 5
      status.storage.replication.factor: 3
      connector.client.config.override.policy: All

    kafka_connect_custom_rest_extension_classes:
      - io.confluent.connect.replicator.monitoring.ReplicatorMonitoringExtension

		kafka_connect_plugins_path:
      - "{{archive_destination_path}}/connect/plugins"

    kafka_connect_plugins_dest: "{{archive_destination_path}}/connect/plugins"
```

- ksql 설정 정보

```
		ksql_log_dir: /logs/ksqldb
    ksql_service_environment_overrides:
      KSQL_HEAP_OPTS: "-Xms1g -Xmx1g"
      KSQL_JVM_PERFORMANCE_OPTS: "-server -XX:+UseG1GC -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35 -XX:+ExplicitGCInvokesConcurrent -XX:MaxInlineLevel=15 -Djava.awt.headless=true"
      KSQL_GC_LOG_OPTS: "-Xlog:gc*:file={{ksql_log_dir}}/ksql-server-gc.log:time,tags:filecount=10,filesize=100M"
      KSQL_JMX_OPTS: "-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false"
      KSQL_LOG4J_OPTS: "-Dlog4j.configuration=file:{{archive_destination_path}}/log4j/ksqldb-log4j.properties"

    ksql_custom_properties:
      ksql.streams.state.dir: /data/ksqldb
```

- control center 설정 정보

```
		control_center_log_dir: /logs/control-center
    control_center_custom_java_args: "-Xlog:gc*:file={{control_center_log_dir}}/control-center-gc.log:time,tags:filecount=10,filesize=
100M"
    control_center_service_environment_overrides:
      CONTROL_CENTER_HEAP_OPTS: "-Xms4g -Xmx4g"
      CONTROL_CENTER_JVM_PERFORMANCE_OPTS: "-server -XX:+UseG1GC -XX:MaxGCPauseMillis=20 -XX:InitiatingHeapOccupancyPercent=35 -XX:+ExplicitGCInvokesConcurrent -XX:MaxInlineLevel=15 -Djava.awt.headless=true"
      CONTROL_CENTER_JMX_OPTS: "-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false"
      CONTROL_CENTER_LOG4J_OPTS: "-Dlog4j.configuration=file:{{archive_destination_path}}/log4j/control-center-log4j.properties"

    control_center_service_overrides:
      Restart: "no"

		control_center_custom_properties:
      confluent.controlcenter.data.dir: /data/control-center
      confluent.controlcenter.command.topic: _confluent-command
      confluent.controlcenter.command.topic.replication: 3
      confluent.controlcenter.internal.topics.partitions: 12
      confluent.controlcenter.internal.topics.replication: 3
      confluent.metrics.topic: _confluent-metrics
      confluent.metrics.topic.partitions: 12
      confluent.metrics.topic.replication: 3
      confluent.controlcenter.streams.num.stream.threads: 8
      confluent.controlcenter.ui.autoupdate.enable: "false"
      confluent.controlcenter.ui.controller.chart.enable: "true"
      confluent.controlcenter.usage.data.collection.enable: "false"
			confluent.controlcenter.id: test01
      confluent.controlcenter.name: test01
```

- Confluent Platform Node 정보

```bash
zookeeper:
  # vars:
  hosts:
    zk01:
      zookeeper_id: 1
    br01:
      zookeeper_id: 2
    sr-ksql01:
      zookeeper_id: 3

kafka_broker:
  # vars:
  hosts:
    zk01:
      broker_id: 1
    br01:
      broker_id: 2
    sr-ksql01:
      broker_id: 3

schema_registry:
  vars:
    schema_registry_custom_properties:
      schema.registry.group.id: schema-cluster1
  hosts:
    sr-ksql01:

kafka_connect:
  children:
    connect-cluster1:
      vars:
        kafka_connect_group_id: connect-cluster1
      hosts:
        cn01:

ksql:
  children:
    ksql-cluster1:
      vars:
        ksql_service_id: ksql-cluster1
      hosts:
        control01:

control_center:
  # vars:
  hosts:
    control01:
```

<aside>
<img src="/icons/forward_lightgray.svg" alt="/icons/forward_lightgray.svg" width="40px" /> **[DOWNLOAD] hosts.yml**

[hosts.yml](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/7fb8f109-a95f-4229-ae8a-ed304e0e9711/hosts.yml)

</aside>

# 4. Install Confluent Platform

### 4-1. 전체 플랫폼 설치 및 기동

```bash
$ ansible-playbook -i hosts.yml confluent.platform.all
```

### 4-2. 컴포넌트별 설치 및 기동

```bash
$ ansible-playbook -i hosts.yml confluent.platform.all --tags=zookeeper
$ ansible-playbook -i hosts.yml confluent.platform.all --tags=kafka_broker
$ ansible-playbook -i hosts.yml confluent.platform.all --tags=schema_registry
$ ansible-playbook -i hosts.yml confluent.platform.all --tags=ksql
$ ansible-playbook -i hosts.yml confluent.platform.all --tags=kafka_connect
$ ansible-playbook -i hosts.yml confluent.platform.all --tags=control_center
```

# 5. Airgapped Deployment : 폐쇄망 환경 내 ansible 설치

폐쇄망 환경에서는 Ansible 설치 및 배포에 필요한 필요한 패키지를 미리 다운로드 받아 준비해야 합니다. 

이 과정에서 별도의 distribution server 가 필요한데, 해당 서버는 ansible-galaxy가 설치되어 있으며, 인터넷 연결이 가능해야 합니다. 

<aside>
💡 Airgapped Deployment 설치 가이드는 폐쇄망 ansible 실습용으로 별도의 디렉토리를 생성하여 파이썬 가상환경(venv) 을 새로 생성하고 시작합니다.

***@ Ansible Control Node***
 *$ mkdir ~/private-ansible ; cd ~/private-ansible
 $ virtualenv --python /usr/bin/python3 venv
 $ source venv/bin/activate*

</aside>

## 5-1. Ansible 설치

distribution server를 통해 Ansible을 설치하기 위해 필요한 패키지를 미리 다운로드 받고,
Ansible-control node에 해당 파일들을 업로드하여 ansible을 설치합니다. 

- ***distribution server***

```bash
*# --- 패키지를 다운받을 디렉토리 준비*
**$ mkdir ~/packages

$ vi ~/requirements.txt**
*virtualenv>=20.16.5
ansible>=2.11*

**$ pip3 download --requirement requirements.txt --dest packages**
...
*Successfully downloaded virtualenv ansible ansible-core distlib filelock importlib-metadata platformdirs resolvelib typing-extensions zipp cryptography jinja2 packaging PyYAML cffi MarkupSafe pyparsing pycparser*

**$ ll ~/packages**
total 57556
-rw-rw-r-- 1 confluent confluent 36832606 Sep 19 13:10 ansible-4.10.0.tar.gz
-rw-rw-r-- 1 confluent confluent  7118115 Sep 19 13:10 ansible-core-2.11.12.tar.gz
-rw-rw-r-- 1 confluent confluent   427911 Sep 19 13:10 cffi-1.15.1-cp37-cp37m-manylinux_2_17_x86_64.manylinux2014_x86_64.whl
-rw-rw-r-- 1 confluent confluent  4143904 Sep 19 13:10 cryptography-38.0.1-cp36-abi3-manylinux_2_17_x86_64.manylinux2014_x86_64.whl
-rw-rw-r-- 1 confluent confluent   468535 Sep 19 13:10 distlib-0.3.6-py2.py3-none-any.whl
-rw-rw-r-- 1 confluent confluent    10057 Sep 19 13:10 filelock-3.8.0-py3-none-any.whl
-rw-rw-r-- 1 confluent confluent    21704 Sep 19 13:10 importlib_metadata-4.12.0-py3-none-any.whl
-rw-rw-r-- 1 confluent confluent   133101 Sep 19 13:10 Jinja2-3.1.2-py3-none-any.whl
-rw-rw-r-- 1 confluent confluent    25334 Sep 19 13:10 MarkupSafe-2.1.1-cp37-cp37m-manylinux_2_17_x86_64.manylinux2014_x86_64.whl
-rw-rw-r-- 1 confluent confluent    40750 Sep 19 13:10 packaging-21.3-py3-none-any.whl
-rw-rw-r-- 1 confluent confluent    14416 Sep 19 13:10 platformdirs-2.5.2-py3-none-any.whl
-rw-rw-r-- 1 confluent confluent   118697 Sep 19 13:10 pycparser-2.21-py2.py3-none-any.whl
-rw-rw-r-- 1 confluent confluent    98338 Sep 19 13:10 pyparsing-3.0.9-py3-none-any.whl
-rw-rw-r-- 1 confluent confluent   596265 Sep 19 13:10 PyYAML-6.0-cp37-cp37m-manylinux_2_5_x86_64.manylinux1_x86_64.manylinux_2_12_x86_64.manylinux2010_x86_64.whl
-rw-rw-r-- 1 confluent confluent    12807 Sep 19 13:10 resolvelib-0.5.4-py2.py3-none-any.whl
-rw-rw-r-- 1 confluent confluent    25596 Sep 19 13:10 typing_extensions-4.3.0-py3-none-any.whl
-rw-rw-r-- 1 confluent confluent  8805743 Sep 19 13:10 virtualenv-20.16.5-py3-none-any.whl
-rw-rw-r-- 1 confluent confluent     5645 Sep 19 13:10 zipp-3.8.1-py3-none-any.whl

*# --- ansible control node에 업로드* 
**$ scp -r packages <control-node>:/home/confluent/private-ansible
$ scp requirements.txt <control-node>:/home/confluent/private-ansible**
```

- ***Ansible Control Node***

반입한 패키지들을 통해 Ansible 및 의존성 라이브러리들을 설치합니다. 

```bash
**(venv) $ cd /home/confluent/private-ansible
(venv) $ pip3 install --no-index --find-links=./packages/ --requirement requirements.txt
*...***
*Successfully built ansible ansible-core*
*****Installing collected packages: resolvelib, distlib, zipp, typing-extensions, PyYAML, pyparsing, pycparser, platformdirs, MarkupSafe, filelock, packaging, jinja2, importlib-metadata, cffi, virtualenv, cryptography, ansible-core, ansible
Successfully installed MarkupSafe-2.1.1 PyYAML-6.0 ansible-4.10.0 ansible-core-2.11.12 cffi-1.15.1 cryptography-38.0.1 distlib-0.3.6 filelock-3.8.0 importlib-metadata-4.12.0 jinja2-3.1.2 packaging-21.3 platformdirs-2.5.2 pycparser-2.21 pyparsing-3.0.9 resolvelib-0.5.4 typing-extensions-4.3.0 virtualenv-20.16.5 zipp-3.8.1*
```

## 5-2. Ansible Playbooks 다운로드 및 Collection Build 작업

distribution server에서 Confluent Platform collection build 한 다음, 해당 collection 을 Ansible-control node에 업로드하여 설치합니다. 

- ***distribution server***

```bash
**$** **git clone https://github.com/confluentinc/cp-ansible.git
$ cd cp-ansible
$ git checkout 7.3.3-post
$ ansible-galaxy collection build**

**## ansible control node에 업로드
****$ scp confluent-platform-7.3.3.tar.gz <control-node>:/home/confluent/private-ansible**
```

- ***Ansible Control Node***

```bash
**$ cd /home/confluent/private-ansible**

*# ---- ansible.cfg 파일 생성*
**$ vi /home/confluent/private-ansible/ansible.cfg** 
*[defaults]
hash_behaviour = merge
collections_paths = **/home/confluent/private-ansible**
host_key_checking = False 
deprecation_warnings=False*

**$ ansible-galaxy collection install confluent-platform-7.3.3.tar.gz**
```

## 5-3. Confluent Platform archive 및 기타 파일 준비

- ***distribution server***

```bash
*# ---  confluent platform archive 파일 다운로드*
**$ curl -O https://packages.confluent.io/archive/7.3/confluent-community-7.3.2.tar.gz**

*# --- monitoring jar 파일 다운로드* 
**$ curl -O https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.17.2/jmx_prometheus_javaagent-0.17.2.jar**

*# --- log4j.properties 파일 업로드*
```

- ***Ansible Control Node***

```bash
*# --- Confluent Componet Node에 배포할 파일들을 위치할 디렉토리 생성*
**$ mkdir /home/confluent/private-ansible/source**
```

- ***distribution server***

```bash
*# --- ansible control node에 배포할 파일들 업로드*
**$ scp confluent-7.3.2.tar.gz <control-node>:/home/confluent/private-ansible/source
$ scp jmx_prometheus_javaagent-0.17.2.jar <control-node>:/home/confluent/private-ansible/source
$ scp *-log4j.properties <control-node>:/home/confluent/private-ansible/source**
```

## 5-4. Configurations for private-ansible

- ***Ansible Control Node : hosts.yml 수정***

```bash
#-- confluent_cli download 비활성화
		*confluent_cli_download_enabled: false*

# -- archvie deployment 방식으로 배포할 수 있도록 설정)
		installation_method: archive
    archive_destination_path: "/engn/confluent"
    confluent_archive_file_remote: false
    *confluent_archive_file_source: "/home/confluent/private-ansible/source/confluent-7.3.2.tar.gz"* 
...
    jmxexporter_enabled: true
    jmxexporter_url_remote: false
    *jmxexporter_jar_url: "/home/confluent/private-ansible/source/jmx_prometheus_javaagent-0.17.2.jar"*
...
		zookeeper_copy_files:
      - *source_path: "/home/confluent/private-ansible/source/log4j/zookeeper-log4j.properties"*
        destination_path: "{{archive_destination_path}}/log4j/zookeeper-log4j.properties"
**
*### ansible-playbook 실행* 
**$ ansible-playbook -i hosts.yml confluent.platform.all**
```

<aside>
<img src="/icons/forward_lightgray.svg" alt="/icons/forward_lightgray.svg" width="40px" /> **[DOWNLOAD] hosts-private.yml**

[hosts-private.yml](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/e4df5a9c-dc34-47f5-b809-14300c972176/hosts-private.yml)

</aside>

# 6. Reconfigure Confluent Platform

Confluent Platform Component 설정 중 변경 사항이 있는 경우 hosts.yml 을 수정한 다음, 
 `--skip-tags package` 을 통해 패키지 설치 task 부분은 건너뛰고 ansible-playbook을 실행한다. 

### 6-1. Update Confluent Platform configuration

```bash
**$ ansible-playbook -i hosts.yml confluent.platform.all --skip-tags package --extra-vars deployment_strategy=rolling**

*...
TASK [confluent.platform.zookeeper : Write Service Overrides] ******************************************************************************
Monday 19 September 2022  14:55:50 +0900 (0:00:00.641)       0:00:37.477 ****** 
--- before: /etc/systemd/system/confluent-zookeeper.service.d/override.conf
+++ after: /home/confluent/.ansible/tmp/ansible-local-56918jugam8jp/tmp_t12ln6s/override.conf.j2
@@ -3,7 +3,7 @@
 # If there is an ExecStart override then we need to clear the ExecStart list first
 ExecStart=
 ExecStart=/engn/confluent/confluent-7.2.1/bin/zookeeper-server-start /engn/confluent/etc/kafka/zookeeper.properties
-Environment="KAFKA_HEAP_OPTS=-Xms256m -Xmx512m"
+Environment="KAFKA_HEAP_OPTS=-Xms512m -Xmx512m"
 Environment="KAFKA_OPTS=-javaagent:/engn/confluent/prometheus/jmx_prometheus_javaagent.jar=1234:/engn/confluent/prometheus/zookeeper.yml"
 Environment="KAFKA_LOG4J_OPTS
...*
```

---

# Appendix A. Trouble Shooting

<aside>
⚠️ **ERROR: Ansible-playbook 실행 시 ImportError: cannot import name 'environmentfilter' from 'jinja2' 발생
>>> 💡 Solution: jinja2를 3.0.1 로 downgrade**
 `$ pip3 install --upgrade jinja2==3.0.1`

</aside>

<aside>
⚠️ **ERROR: Ansible-playbook 실행 시 “debug3: mux_client_read_packet: read header failed: Broken pipe” 발생
>>> 💡 Solution: 에러가 발생한 특정 yml 파일에서 environmentfilter 부분 수정**

```bash
1. *from jinja2.filters import ~~environmentfilter~~* >> 
**try:    from jinja2.filters import environmentfilter as pass_environment
except ImportError: # renamed in jinja2 3.1
    from jinja2.filters import pass_environment**

2. *~~@environmentfilter~~ >>*  **@pass_environment**  
```

</aside>

# Appendix. B: Configure TLS Encryption

### B-1. 각 서버별 Keystore 및  Truststore 파일 준비

: 아래 절차에 따라 각 Confluent Platform Nodes 상에 Keystore를 구성하여 준비한다. 

```bash
# ------- 모든 Platform Nodes가 동일하게 사용할 Root CA 와 Truststore 구성 

### Crate the Root CA : private key & root CA 생성
openssl genrsa -out root.key
openssl req -new -x509 -key root.key -out root.crt

### set permission for files
chmod 600 root.key
chmod 644 root.crt

# ------- Configure Truststroe file for all the Nodes 
keytool -keystore kafka.truststore.p12 -alias CARoot -import  -file root.crt  -storetype pkcs12
scp kafka.truststore root.key  root.crt  confluent@<ansible-control-node>:~/ssl

# ------- 아래 절차는 각 node에 맞게 hostname 다르게 하여 수행 (ex: "zk01" node) 
### Configure Unique Keystore for Platform Nodes 
keytool -keystore zk01.keystore.p12 -alias localhost -validity 365 -genkey -keyalg RSA  -ext san=dns:zk01 -storetype pkcs12

### Create signed crt 
keytool -keystore zk01.keystore.p12 -alias localhost  -certreq -file zk01.unsigned.crt
openssl x509 -req -CA root.crt -CAkey root.key -in zk01.unsigned.crt  -out zk01.signed.crt -days 365 -CAcreateserial
"Signature ok
subject=/C=KR/ST=SEOUL/L=SEOUL/O=TG/OU=DS/CN=SONYE
Getting CA Private Key"

### import rootCa > Keystore 
keytool -keystore zk01.keystore.p12  -alias CARoot -import -file root.crt
"Certificate was added to keystore"

### import signedCrt > Keystore 
keytool -keystore zk01.keystore.p12   -alias localhost    -import -file zk01.signed.crt 
keytool -keystore br01.keystore.p12 -alias localhost    -import -file br01.signed.crt 
keytool -keystore sr-ksql01.keystore.p12  -alias localhost    -import -file sr-ksql01.signed.crt 
keytool -keystore cn01.keystore.p12  -alias localhost    -import -file cn01.signed.crt 
keytool -keystore control01.keystore.p12  -alias localhost    -import -file control01.signed.crt 
"Certificate reply was installed in keystore"
```

### B-2. hosts.yml 수정

- SSL 관련 설정 추가:

```bash
---
all:
  vars:
    ssl_enabled: true
    ssl_mutual_auth_enabled: true
    ssl_provided_keystore_and_truststore: true
    ssl_provided_keystore_and_truststore_remote_src: true
```

- broker listener 추가:

```bash
    ### kafka listeners
    kafka_broker_configure_multiple_listeners: true
    kafka_broker_configure_control_plane_listener: false
    kafka_broker_inter_broker_listener_name: broker
    kafka_broker_custom_listeners:
      internal:
        name: INTERNAL
        port: 9092
        ssl_enabled: true
        sasl_protocol: none
      broker:
        name: BROKER
        port: 9093
        ssl_enabled: true
        sasl_protocol: none
```

- 각 component 별 keystore/truststore 관련 정보 명시:

```bash
zookeeper:
  # vars:
  hosts:
    zk01:
      zookeeper_id: 1
      ssl_keystore_filepath: "/home/confluent/ssl/zk01.keystore.p12"
      ssl_keystore_key_password: test1234
      ssl_keystore_store_password: test1234
      ssl_keystore_alias: localhost
      ssl_truststore_filepath: "/home/confluent/ssl/kafka.truststore.p12"
      ssl_truststore_password: test1234
      ssl_truststore_ca_cert_alias: caroot
```

### B-3. 결과

- Zookeeper/Broker 서버에 생성된 client.properties

```bash
[confluent@zk01 kafka]$ cat zookeeper-tls-client.properties
# Maintained by Ansible
zookeeper.clientCnxnSocket=org.apache.zookeeper.ClientCnxnSocketNetty
zookeeper.ssl.client.enable=true
zookeeper.ssl.truststore.location=/home/confluent/ssl/kafka.truststore.p12
zookeeper.ssl.truststore.password=test1234
zookeeper.ssl.keystore.location=/home/confluent/ssl/zk01.keystore.p12
zookeeper.ssl.keystore.password=test1234

[confluent@zk01 kafka]$ cat client.properties
# Maintained by Ansible
# Note: Secrets file for decryption when secrets protection is enabled can be found in /var/ssl/private/kafka-broker-client-security.properties
security.protocol=SSL
ssl.key.password=test1234
ssl.keystore.location=/home/confluent/ssl/zk01.keystore.p12
ssl.keystore.password=test1234
ssl.truststore.location=/home/confluent/ssl/kafka.truststore.p12
ssl.truststore.password=test1234
default.api.timeout.ms=20000
request.timeout.ms=20000
```

- Broker 기동 로그 확인:

```bash
[2023-04-18 15:58:37,653] INFO [Admin Manager on Broker 1]: **User:CN=sr-ksql01,OU=DS,O=TG,L=SEOUL,ST=SEOUL,C=KR** is updating topic _confluent_balancer_partition_samples with new configuration : cleanup.policy -> delete,retention.ms -> 3600000 (kafka.server.ZkAdminManager)
```

<aside>
<img src="/icons/forward_lightgray.svg" alt="/icons/forward_lightgray.svg" width="40px" /> **[DOWNLOAD] hosts-ssl.yml**

[hosts-ssl.yml](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/0dd8bad1-b38e-4ef4-b367-8a97fb813ccd/hosts-ssl.yml)

</aside>

# Appendix. C : **Configure SASL/GSSAPI (Kerberos) authentication**

### C-1. 각 서버별 Kerberos keytab 파일 준비

: 아래 절차에 따라 Kerberos 서버에서 생성한 keytab파일을 Ansible Control Node로 옮겨온다.

```bash
@ kerberos 서버

# ------- principal 추가 

###  add_principal for zookeeper service
sudo kadmin.local -q "add_principal -randkey zookeeper/zk01@KAFKA.SECURE"
sudo kadmin.local -q "add_principal -randkey zookeeper/br01@KAFKA.SECURE"
sudo kadmin.local -q "add_principal -randkey zookeeper/sr-ksql01@KAFKA.SECURE"

###  add_principal for broker service 
sudo kadmin.local -q "add_principal -randkey kafka/zk01@KAFKA.SECURE"
sudo kadmin.local -q "add_principal -randkey kafka/br01@KAFKA.SECURE"
sudo kadmin.local -q "add_principal -randkey kafka/sr-ksql01@KAFKA.SECURE"

###  add_principal for other components
sudo kadmin.local -q "add_principal -randkey schemaregistry/sr-ksql01@KAFKA.SECURE"
sudo kadmin.local -q "add_principal -randkey connect/cn01@KAFKA.SECURE"
sudo kadmin.local -q "add_principal -randkey ksql/control01@KAFKA.SECURE"
sudo kadmin.local -q "add_principal -randkey controlcenter/control01@KAFKA.SECURE"

# ------- keytab 파일 생성

### export principal to keytab files : zookeeper service 
sudo kadmin.local -q "xst -kt /home/confluent/keytabs/zookeeper_zk01.service.keytab zookeeper/zk01@KAFKA.SECURE"
sudo kadmin.local -q "xst -kt /home/confluent/keytabs/zookeeper_br01.service.keytab zookeeper/br01@KAFKA.SECURE"
sudo kadmin.local -q "xst -kt /home/confluent/keytabs/zookeeper_sr-ksql01.service.keytab zookeeper/sr-ksql01@KAFKA.SECURE"

### export principal to keytab files : broker service 
sudo kadmin.local -q "xst -kt /home/confluent/keytabs/kafka_zk01.service.keytab kafka/zk01@KAFKA.SECURE"
sudo kadmin.local -q "xst -kt /home/confluent/keytabs/kafka_br01.service.keytab kafka/br01@KAFKA.SECURE"
sudo kadmin.local -q "xst -kt /home/confluent/keytabs/kafka_sr-ksql01.service.keytab kafka/sr-ksql01@KAFKA.SECURE"

### export principal to keytab files : other components 
sudo kadmin.local -q "xst -kt /home/confluent/keytabs/schema01.service.keytab schemaregistry/sr-ksql01@KAFKA.SECURE"
sudo kadmin.local -q "xst -kt /home/confluent/keytabs/connect01.service.keytab connect/cn01@KAFKA.SECURE"
sudo kadmin.local -q "xst -kt /home/confluent/keytabs/ksql01.service.keytab ksql/control01@KAFKA.SECURE"
sudo kadmin.local -q "xst -kt /home/confluent/keytabs/controlcenter.service.keytab controlcenter/control01@KAFKA.SECURE"

# ------ ansible 기동 user 가 해당 keytab파일을 읽을 수 있도록 권한 조정
sudo chown -R confluent:confluent *

scp *.keytab confluent@<ansible-control-node>:~/keytabs
```

### C-2. hosts.yml 수정

- sasl_protocol 명시 및 kerberos 관련 설정 추가:

```bash
---
all:
  vars:
    **kerberos_configure: true
    kerberos:
      realm: KAFKA.SECURE ### realm configured on kerberos server 
      kdc_hostname: control02 ### kerberos server host name 
      admin_hostname: control02 ### kerberos server host name**
```

- sasl_protocol 명시 및 broker listener 설정 추가:

```bash
		### sasl authentication
    sasl_protocol: **kerberos**

		### kafka listeners
		kafka_broker_custom_listeners:
      internal:
        name: INTERNAL
        port: 9092
        ssl_enabled: false
        sasl_protocol: **kerberos**
      broker:
        name: BROKER
        port: 9093
        ssl_enabled: false
        sasl_protocol: **kerberos**
```

- 각 component 별 keytabs 파일 경로 및 principal 명시:

```bash
zookeeper:
  # vars:
  hosts:
    zk01:
			zookeeper_kerberos_keytab_path: /home/confluent/keytabs/zookeeper_zk01.keytab 
      zookeeper_kerberos_principal: zookeeper/zk01@KAFKA.SECURE
```

### C-3. 결과

- 각 컴포넌트에 생성된 kerberos client configuration

```bash
## kerberos_configure 설정 값에 따라 작성된 kerberos_client_configuration file (default: /etc/krb5.conf)
 
[confluent@zk01 keytabs]$ cat /etc/krb5.conf
[libdefaults]
 default_realm = KAFKA.SECURE
 dns_lookup_realm = false
 dns_lookup_kdc = false
 ticket_lifetime = 24h
 forwardable = true
 udp_preference_limit = 1
 default_tkt_enctypes = aes256-cts-hmac-sha1-96 aes128-cts-hmac-sha1-96 arc-four-hmac rc4-hmac
 default_tgs_enctypes = aes256-cts-hmac-sha1-96 aes128-cts-hmac-sha1-96 arc-four-hmac rc4-hmac
 permitted_enctypes = aes256-cts-hmac-sha1-96 aes128-cts-hmac-sha1-96 arc-four-hmac rc4-hmac
 canonicalize = True

[realms]
 KAFKA.SECURE = {
  kdc = control02:88
  admin_server = control02:749
  default_domain = kafka.secure
 }

[domain_realm]
 .kafka.secure = KAFKA.SECURE
  kafka.secure = KAFKA.SECURE
```

- Broker 서버에 생성된 client.properties

```bash
[confluent@zk01 kafka]$ cat client.properties
# Maintained by Ansible
# Note: Secrets file for decryption when secrets protection is enabled can be found in /var/ssl/private/kafka-broker-client-security.properties
sasl.jaas.config=com.sun.security.auth.module.Krb5LoginModule required useKeyTab=true storeKey=true keyTab="/etc/security/keytabs/kafka_broker.keytab" principal="kafka/zk01@KAFKA.SECURE";
sasl.kerberos.service.name=kafka
sasl.mechanism=GSSAPI
security.protocol=SASL_PLAINTEXT
default.api.timeout.ms=20000
request.timeout.ms=20000
```

- Broker 기동 로그 확인:

```bash
[2023-03-20 10:57:09,407] INFO Successfully authenticated client: authenticationID=kafka/zk01@KAFKA.SECURE; authorizationID=kafka/zk01@KAFKA.SECURE. (org.apache.kafka.common.security.authenticator.SaslServerCallbackHandler)
[2023-03-20 10:57:09,471] INFO Successfully authenticated client: authenticationID=kafka/sr-ksql01@KAFKA.SECURE; authorizationID=kafka/sr-ksql01@KAFKA.SECURE. (org.apache.kafka.common.security.authenticator.SaslServerCallbackHandler)
[2023-03-20 10:57:09,498] INFO Successfully authenticated client: authenticationID=kafka/br01@KAFKA.SECURE; authorizationID=kafka/br01@KAFKA.SECURE. (org.apache.kafka.common.security.authenticator.SaslServerCallbackHandler)
...

[2023-03-20 11:07:38,577] INFO Successfully authenticated client: authenticationID=controlcenter/control01@KAFKA.SECURE; authorizationID=controlcenter/control01@KAFKA.SECURE. (org.apache.kafka.common.security.authenticator.SaslServerCallbackHandler)
[2023-03-20 11:08:06,118] INFO Successfully authenticated client: authenticationID=ksql/control01@KAFKA.SECURE; authorizationID=ksql/control01@KAFKA.SECURE. (org.apache.kafka.common.security.authenticator.SaslServerCallbackHandler)
```

<aside>
<img src="/icons/forward_lightgray.svg" alt="/icons/forward_lightgray.svg" width="40px" /> **[DOWNLOAD] hosts-kerberos.yml**

[hosts-kerberos.yml](https://s3-us-west-2.amazonaws.com/secure.notion-static.com/8502f44d-fafb-4eb3-96ca-9458885459a3/hosts-kerberos.yml)

</aside>

# Appendix. D : Ansible 설치시 서비스 자동 시작 비활성화

### D-1. hosts.yml 수정

```bash
---
all:
  vars:
    **health_checks_enabled: false <<추가** 
```

### D-2. roles/${COMPONENT_NAME}/tasks/main.yml 수정

- main.yml 에서 Service Start 하는 task 부분의 tags를 systemd_start 로 수정

```bash

- name: Zookeeper Service Started
  systemd:
    name: "{{zookeeper_service_name}}"
    enabled: true
    state: started
  tags:
    - systemd_start
```

### D-3. skip-tags 옵션을 통해 PLAYBOOK 실행

```bash
ansible-playbook  -i hosts.yml  confluent.platform.all --tag zookeeper --skip-tags systemd_start
```

# Reference.

- Ansible Playbooks for Confluent Platform
    
    [Ansible Playbooks for Confluent Platform](https://docs.confluent.io/ansible/current/overview.html)
    

- confluent platform 설치용 hosts.yml에 사용되는 role variables 별 설명
    
    [cp-ansible/VARIABLES.md at 7.3.3-post · confluentinc/cp-ansible](https://github.com/confluentinc/cp-ansible/blob/7.3.3-post/docs/VARIABLES.md)