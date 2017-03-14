```sh
#-------
# vagrant
#-------
    aria2c  -c -x 16 https://atlas.hashicorp.com/ubuntu/boxes/trusty64/versions/20160708.1.1/providers/virtualbox.box
#------

#-------
# machine setup
#-------
    # /etc/network/interfaces
    # iface em1 inet static
    # address 192.168.1.?
    # netmask 255.255.0.0
    # gateway 192.168.1.1
    # dns-nameservers 192.168.1.1 8.8.8.8
    # sudo ifdown em1
    # sudo ifup em1

    if ! sudo grep 'spiderdt' /etc/sudoers > /dev/null; then echo "spiderdt ALL=(ALL) NOPASSWD: ALL" | sudo tee -a /etc/sudoers; fi

    # cat /dev/zero | ssh-keygen -q -N ""
    mkdir -p ~/.ssh
    wget -O ~/.ssh/id_rsa https://raw.githubusercontent.com/clojurians-org/spiderdt-env/master/home/.ssh/id_rsa
    chmod 700 ~/.ssh/id_rsa
    wget -O ~/.ssh/id_rsa.pub https://raw.githubusercontent.com/clojurians-org/spiderdt-env/master/home/.ssh/id_rsa.pub
    cat ~/.ssh/id_rsa.pub > ~/.ssh/authorized_keys

    sudo apt-get update
    sudo apt-get install openssh-server

#--------
# ansible docker
#--------
    #--------
    # prepare file
    #--------
        cp  ~spiderdt-ftp/data-platform/tarball/python2.7.tar tarball
    #--------
    # install ansible
    #--------
        # curl -sSL https://bootstrap.pypa.io/get-pip.py | sudo -H python
        cp  tarball/python2.7.tar /usr/local/lib/python2.7
        tar -xvf python2.7.tar
        sudo cp tarball/pip /usr/local/bin/pip
        sudo apt-get install python-dev libssl-dev
        sudo pip install ansible
    #--------

    #--------
    # client configuration
    #--------
        for group in data-ftp data-platform data-service data-ui; do
            ansible client -s -i hosts -m group -a "name=$group state=present"
        done

        echo "root:spiderdt" | sudo chpasswd
        ansible client -s -i hosts -m user -a "name=hive group=data-platform groups=data-platform"
        #=> data-platform[docker] user
        for user in root larluo steve chong kun dll; do
            ansible client -s -i hosts -m user -a "name=$user group=data-platform groups=data-platform,docker shell=/bin/bash append=yes"
            # echo "$user:spiderdt"  | sudo chpasswd
            ansible client -s -i hosts -m copy -a "src=~spiderdt/.ssh dest=~$user owner=$user group=data-platform"
            ansible client -s -i hosts -m file -a "path=~$user/.ssh/id_rsa mode=0700"
            ansible client -s -i hosts -m file -a "path=~$user/work state=directory mode=770 owner=$user"
            ansible client -s -i hosts -m file -a "src=~spiderdt/work/git dest=~$user/work/git state=link owner=$user"
        done

    #--------

    #--------
    # install docker
    #--------
    # https://docs.docker.com/engine/installation/linux/ubuntulinux/
    ansible docker -s -i hosts -m copy -a "src=root/etc/apt/sources.list.d/docker.list dest=/etc/apt/sources.list.d/docker.list mode=0644"
    ansible docker -s -i hosts -m apt -a "name=apt-transport-https state=present"
    ansible docker -s -i hosts -m apt -a "name=ca-certificates state=present"
    ansible docker -s -i hosts -m command -a "apt-key adv --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys"
    ansible docker -s -i hosts -m apt -a "update_cache=yes"
    ansible docker -s -i hosts -m apt -a "name=docker-engine state=present force=yes"
    ansible docker -i hosts -m service -a "name=docker state=started"
    ansible docker -s -i hosts -m group -a "name=docker state=present"
    ansible docker -s -i hosts -m user -a "name=spiderdt groups=docker append=yes"

    # => install docker-compose
    # curl -L https://github.com/docker/compose/releases/download/1.6.2/docker-compose-`uname -s`-`uname -m`  > a.sh
    ansible docker -s -i hosts -m copy -a "src=tarball/docker-compose dest=/usr/local/bin/docker-compose mode=0755"

    # => install pip
    ansible client_cluster -i hosts -m raw -a "curl -sSL https://bootstrap.pypa.io/get-pip.py | sudo -H python"
    # ansible cluster -s -i hosts -m apt -a "name=libssl-dev state=present force=yes"
    # ansible cluster -s -i hosts -m apt -a "name=python-dev state=present force=yes"
    ansible cluster -s -i hosts -m copy -a "src=tarball/python2.7.tar dest=/usr/local/lib/python2.7"
    ansible cluster -s -i hosts -m raw -a "cd /usr/local/lib/python2.7 && tar -xf python2.7.tar"
    ansible cluster -s -i hosts -m copy -a "src=tarball/pip dest=/usr/local/bin/pip mode=0755"

    # => preapre docker build
    ansible client_cluster -i hosts -m file -a "path=work/git/spiderdt-env/docker state=directory mode=0755"
    ansible client -i hosts -m file -a "path=work/git/spiderdt-env/cluster/tarball state=directory mode=0755"
    ansible client -i hosts -m copy -a "src=docker/ubuntu#14.04.docker dest=work/git/spiderdt-env/cluster/docker mode=0644"
    ansible client -i hosts -m raw -a "docker load < work/git/spiderdt-env/cluster/docker/ubuntu#14.04.docker"
    # wget -c tarball/jdk-8u92-linux-x64.tar.gz --no-check-certificate --no-cookies --header "Cookie: oraclelicense=accept-securebackup-cookie" http://download.oracle.com/otn-pub/java/jdk/8u92-b14/jdk-8u92-linux-x64.tar.gz
    ansible client -i hosts -m copy -a "src=tarball/jdk-8u92-linux-x64.tar.gz dest=work/git/spiderdt-env/cluster/tarball mode=0644"
    ansible client -i hosts -m copy -a "src=tarball/zookeeper-3.4.8.tar.gz dest=work/git/spiderdt-env/cluster/tarball mode=0644"
    ansible client -i hosts -m copy -a "src=tarball/hadoop-2.7.2.tar.gz dest=work/git/spiderdt-env/cluster/tarball mode=0644"
#--------

#--------
# cluster ip->name
#--------
    ansible all -s -i hosts -m copy -a "src=root/etc/hosts dest=/etc/hosts mode=0644"
#--------

#--------
# zookeeper docker cluster
#--------
    # => share host information
    ansible client_cluster -s -i hosts -m file -a "path=/data/zookeeper/docker_host state=directory owner=spiderdt group=spiderdt mode=0755"
    ansible client_cluster -i hosts -m raw -a "echo '# ifconfig' > /data/zookeeper/docker_host/ifconfig.out && ifconfig >> /data/zookeeper/docker_host/ifconfig.out"

    # => load docker image
    ansible cluster -s -i hosts -m file -a "path=work/git/spiderdt-env/docker state=directory owner=spiderdt group=spiderdt mode=0755"
    ansible cluster -i hosts -m copy -a "src=docker/spiderdt#zookeeper#1.0.docker dest=work/git/spiderdt-env/docker mode=0644"
    ansible cluster -i hosts -m raw -a "docker load < work/git/spiderdt-env/docker/spiderdt#zookeeper#1.0.docker"
    ansible cluster -i hosts -m shell -a "rm -rf  work/git/spiderdt-env/docker/spiderdt#zookeeper#1.0.docker"

    # => run in cluster mode
    ansible cluster -i hosts -m docker -a "name=zookeeper image=spiderdt/zookeeper:1.0 detach=true net=host state=restarted restart_policy=always env='{\"SERVERS\":\"192.168.1.3,192.168.1.4,192.168.1.5,192.168.1.6,192.168.1.7,192.168.1.8,192.168.1.9\"}' volumes='/data/zookeeper:/data/zookeeper'"

#--------
# hadoop-hdfs docker cluster
#--------
    # => share host information
    ansible client_cluster -s -i hosts -m file -a "path=/data/hdfs/docker_host state=directory owner=spiderdt group=spiderdt mode=0770"
    ansible client_cluster -s -i hosts -m file -a "path=/data/hdfs/namenode state=directory owner=spiderdt group=spiderdt mode=0770"
    ansible client_cluster -s -i hosts -m file -a "path=/data/hdfs/datanode state=directory owner=spiderdt group=spiderdt mode=0770"
    ansible client_cluster -i hosts -m raw -a "echo '# ifconfig' > /data/hdfs/docker_host/ifconfig.out && ifconfig >> /data/hdfs/docker_host/ifconfig.out"

    # => load docker image
    ansible cluster -i hosts -m copy -a "src=docker/spiderdt#hdfs#1.0.docker dest=work/git/spiderdt-env/docker mode=0644"
    ansible cluster -i hosts -m raw -a "docker load < work/git/spiderdt-env/docker/spiderdt#hdfs#1.0.docker"
    ansible cluster -i hosts -m shell -a "rm -rf  work/git/spiderdt-env/docker/spiderdt#hdfs#1.0.docker"

    # => run in cluster mode
    ansible cluster -i hosts -m docker -a "name=hdfs image=spiderdt/hdfs:1.0 detach=true net=host state=restarted restart_policy=always env='{\"MASTER\":\"192.168.1.3\"}' volumes='/data/hdfs:/data/hdfs'"

#--------
# hadoop-yarn docker cluster
#--------
    # => share host information
    ansible client_cluster -s -i hosts -m file -a "path=/data/yarn/docker_host state=directory owner=spiderdt group=spiderdt mode=0755"
    ansible client_cluster -s -i hosts -m file -a "path=/data/yarn/nodemanager state=directory owner=spiderdt group=spiderdt mode=0755"
    ansible client_cluster -i hosts -m raw -a "echo '# ifconfig' > /data/yarn/docker_host/ifconfig.out && ifconfig >> /data/yarn/docker_host/ifconfig.out"

    # => load docker image
    ansible cluster -i hosts -m shell -a "docker stop yarn"
    ansible cluster -i hosts -m shell -a "docker rm yarn"
    ansible cluster -i hosts -m shell -a "docker rmi spiderdt/yarn:1.0"
    ansible cluster -i hosts -m copy -a "src=docker/spiderdt#yarn#1.0.docker dest=work/git/spiderdt-env/docker mode=0644"
    ansible cluster -i hosts -m raw -a "docker load < work/git/spiderdt-env/docker/spiderdt#yarn#1.0.docker"
    ansible cluster -i hosts -m shell -a "rm -rf  work/git/spiderdt-env/docker/spiderdt#yarn#1.0.docker"

    # => run in cluster mode
    ansible cluster -i hosts -m docker -a "name=yarn image=spiderdt/yarn:1.0 detach=true net=host state=restarted restart_policy=always env='{\"MASTER\":\"192.168.1.3\"}' volumes='/data/yarn:/data/yarn'"
#--------

#--------
# hbase docker cluster
#--------
    # => share host information
    ansible client_cluster -s -i hosts -m file -a "path=/data/hbase/docker_host state=directory owner=spiderdt group=spiderdt mode=0755"
    ansible client_cluster -i hosts -m raw -a "echo '# ifconfig' > /data/hbase/docker_host/ifconfig.out && ifconfig >> /data/hbase/docker_host/ifconfig.out"

    # => load docker image
    ansible cluster -i hosts -m copy -a "src=docker/spiderdt#hbase#1.0.docker dest=work/git/spiderdt-env/cluster/docker mode=0644"
    ansible cluster -i hosts -m raw -a "docker load < work/git/spiderdt-env/cluster/docker/spiderdt#hbase#1.0.docker"
    ansible cluster -i hosts -m shell -a "rm -rf  work/git/spiderdt-env/cluster/docker/spiderdt#hbase#1.0.docker"

    # => run in cluster mode
    ansible cluster -i hosts -m docker -a "name=hbase image=spiderdt/hbase:1.0 detach=true net=host state=restarted restart_policy=always env='{\"MASTER\":\"192.168.1.3\",\"ZK_SERVERS\":\"192.168.1.3,192.168.1.4,192.168.1.5,192.168.1.6,192.168.1.7,192.168.1.8,192.168.1.9\"}' volumes='/data/hbase:/data/hbase'"

#--------
# postgres docker
#--------
    #=> load docker image
    ansible core-service -i hosts -m copy -a "src=docker/spiderdt#pgsql#1.0.docker dest=work/git/spiderdt-env/docker mode=0644"
    ansible core-service -i hosts -m raw -a "docker load < work/git/spiderdt-env/docker/spiderdt#pgsql#1.0.docker"
    ansible core-service -i hosts -m shell -a "rm -rf  work/git/spiderdt-env/docker/spiderdt#pgsql#1.0.docker"

    #=> run in cluster mode
    ansible core-service -i hosts -m docker -a "name=pgsql image=spiderdt/pgsql:1.0 detach=true net=host state=restarted restart_policy=always volumes='/data/pgsql:/data/pgsql'"
#--------
# client setup
#--------
    #=> download tool
    ansible client -s -i hosts -m apt -a "name=aria2 state=present"
    ansible client -s -i hosts -m apt -a "name=tree state=present"
    ansible client -s -i hosts -m apt -a "name=unzip state=present"

    #=> java jdk8
    ansible client -s -i hosts -m copy -a "src=~spiderdt-ftp/data-platform/tarball/jdk-8u60-linux-x64.tar.gz dest=~spiderdt/work/git/spiderdt-env/tarball/jdk-8u60-linux-x64.tar.gz owner=spiderdt group=data-platform"


    # prioxy
    sudo sslocal -c conf/shadowsocks/config.json

    #=> hdfs
    ansible client -s -i hosts -m raw -a ". ~/.profile && \$HADOOP_PREFIX/bin/hdfs dfs -mkdir -p /\{user,tmp}"
    ansible client -s -i hosts -m raw -a ". ~/.profile && \$HADOOP_PREFIX/bin/hdfs dfs -chmod 777 /user"
    ansible client -s -i hosts -m raw -a ". ~/.profile && \$HADOOP_PREFIX/bin/hdfs dfs -chmod 777 /tmp"
#--------

#--------
# cheet sheet
#--------
    # ansible all -i hosts -m command -a  "hostname"
    # ansible all -i hosts -m command -a  "more /etc/issue"

    # docker build -t spiderdt/zookeeper:1.0 .
    # docker run -it -e SERVERS=192.168.1.3,192.168.1.4,192.168.1.5 -v /data/zookeeper:/data/zookeeper spiderdt/zookeeper:1.0 bash
    # docker save spiderdt/zookeeper:1.0 > ~spiderdt/work/git/spiderdt-env/docker/spiderdt#zookeeper#1.0.docker

    # docker build -t spiderdt/yarn:1.0 .
    # docker run -it --name "yarn" --net=host -e MASTER=192.168.1.3 -v /data/yarn:/data/yarn spiderdt/yarn:1.0 bash
    # docker run -d --name "yarn" --net=host -e MASTER=192.168.1.3 -v /data/yarn:/data/yarn spiderdt/yarn:1.0 -d
    # docker save spiderdt/yarn:1.0 > ~spiderdt/work/git/spiderdt-env/docker/spiderdt#yarn#1.0.docker

    # docker build -t spiderdt/hbase:1.0 .
    # docker run -it --net=host -e MASTER=192.168.1.3 -e ZK_SERVERS=192.168.1.3,192.168.1.4,192.168.1.5 -v /data/hbase:/data/hbase spiderdt/hbase:1.0 bash
    # docker save spiderdt/hbase:1.0 > ~spiderdt/work/git/spiderdt-env/docker/spiderdt#hbase#1.0.docker

    # ansible all -i hosts -m raw -a "docker stop zookeeper"
    # ansible all -i hosts -m raw -a "docker rm zookeeper"
    # ansible all -i hosts -m raw -a "docker stop \$(docker ps -a -q)"
    # ansible all -i hosts -m raw -a "docker rmi \$(docker images -q)"

    docker save spiderdt/pgsql:1.0 > ~spiderdt/work/git/spiderdt-env/docker/spiderdt#pgsql#1.0.docker

    # brew cask install google-chrome
    nohup sslocal -c conf/shadowsocks/config.json > /dev/null 2>&1 &
    curl --socks5 127.0.0.1:1080 http://www.google.com
    curl --proxy http://127.0.0.1:8118 http://www.google.com
    /etc/init.d/privoxy start

#---------
# Build
#--------
  # git lfs
    curl -s https://packagecloud.io/install/repositories/github/git-lfs/script.deb.sh | sudo bash
    sudo apt-get install git-lfs

#--------

#---------
# core-service
#--------
    docker save spiderdt/yarn:1.0 > ~spiderdt/work/git/spiderdt-env/docker/spiderdt#yarn#1.0.docker
    ansible cluster -i hosts -m copy -a "src=docker/spiderdt#yarn#1.0.docker dest=work/git/spiderdt-env/docker mode=0644"
    ansible cluster -i hosts -m raw -a "docker load < work/git/spiderdt-env/docker/spiderdt#yarn#1.0.docker"
    ansible cluster -i hosts -m shell -a "rm -rf  work/git/spiderdt-env/docker/spiderdt#yarn#1.0.docker"

    # hive
    hive --hiveconf hive.root.logger=TRACE,console
    hive --service hiveserver2 --hiveconf hive.root.logger=TRACE,console
    nohup hive --service hiveserver2  2>&1 > /dev/null &

    # mongo
    docker run -d --name mongo -v /data/mongo:/data/db -p '0.0.0.0:27017:27017' mongo

    #=> redis server
    ansible client -s -i hosts -m file -a "path=/data/redis state=directory owner=spiderdt group=spiderdt mode=0755"
    ansible client -i hosts -m docker -a "name=redis image=redis:3.2.3 ports='0.0.0.0:6379:6379' state=restarted restart_policy=always volumes='/data/redis:/data'"

    #=> postgres server
    ./pg_ctl -D /data/pgsql -l logfile start
    ./createdb postgres
    ./dropuser postgres
    ./createuser -s postgres -P
    ./psql --username postgres
    CREATE ROLE ms LOGIN PASSWORD 'spiderdt' CREATEDB VALID UNTIL 'infinity' ;

    # chronos
    nohup /home/spiderdt/work/git/spiderdt-env/tarball/mesos/build/bin/mesos-master.sh --ip=192.168.1.2 --work_dir=/data/mesos 2>&1 > /dev/null &
    nohup /home/spiderdt/work/git/spiderdt-env/tarball/mesos/build/bin/mesos-slave.sh --master=192.168.1.2:5050 --switch_user=false --work_dir=/data/mesos 2>&1 > /dev/null &
    export MESOS_NATIVE_LIBRARY=/usr/local/lib/libmesos.so
    nohup java -cp /home/spiderdt/work/git/spiderdt-env/tarball/chronos/target/chronos*.jar org.apache.mesos.chronos.scheduler.Main --master 192.168.1.2:5050 --zk_hosts 192.168.1.3:2181 2>&1 > /dev/null &

    # ssl [/usr/lib/ssl/openssl.cnf]
    openssl genrsa -aes256 -out /data/ssl/ca/ca.key.pem 4096 [spiderdt.com]
    openssl req -key /data/ssl/ca/ca.key.pem -new -x509 -days 7300 -sha256 -out /data/ssl/ca/ca.cert.pem
    [info] openssl x509 -noout -text -in /data/ssl/ca/ca.cert.pem
    openssl genrsa -aes256 -out /data/ssl/server/server.key.pem.pass 2048 [fill]
    openssl rsa -in /data/ssl/server/server.key.pem.pass -out /data/ssl/server/server.key.pem
    openssl req -key /data/ssl/server/server.key.pem -new -sha256 -out /data/ssl/server/server.csr.pem[fill]
    [sign server]openssl ca -config /data/ssl/ca/openssl.cnf -keyfile /data/ssl/ca/ca.key.pem -cert /data/ssl/ca/ca.cert.pem -extensions server_cert -days 375 -notext -md sha256 -in /data/ssl/server/server.csr.pem -out /data/ssl/server/server.cert.pem
    [info] openssl x509 -noout -text -in /data/ssl/server/server.cert.pem
    openssl genrsa -aes256 -out /data/ssl/client/client.key.pem.pass 2048 [fill]
    openssl rsa -in /data/ssl/client/client.key.pem.pass -out /data/ssl/client/client.key.pem
    openssl req -key /data/ssl/client/client.key.pem -new -sha256 -out /data/ssl/client/client.csr.pem[fill]
    [sign client]openssl ca -config /data/ssl/ca/openssl.cnf -keyfile /data/ssl/ca/ca.key.pem -cert /data/ssl/ca/ca.cert.pem -extensions client_cert -days 375 -notext -md sha256 -in /data/ssl/client/client.csr.pem -out /data/ssl/client/client.cert.pem
    chmod 600 /data/ssl/server/server.key.pem
    chmod 600 /data/ssl/client/client.key.pem
    cp /data/ssl/ca/ca.cert.pem /data/ssl/client/root.cert.pem
    cp /data/ssl/ca/ca.cert.pem /data/ssl/server/root.cert.pem
    openssl pkcs8 -topk8 -inform PEM -outform DER -in /data/ssl/client/client.key.pem -out /data/ssl/client/client.key.pk8 -nocrypt

    /home/spiderdt/work/git/spiderdt-release/opt/pgsql/bin/psql "sslmode=require host=192.168.1.2 dbname=ms" -U ms

    http://kr.archive.ubuntu.com/ubuntu/pool/main/g/

    sudo apt-get install python-software-properties
    sudo apt-add-repository ppa:libreoffice/ppa
    sudo apt update

If you are running from maven, you need to use the additionalparam setting, as per the manual. Either add it as a global property:
  <properties>
    <additionalparam>-Xdoclint:none</additionalparam>
  </properties>



```
