---
layout: post
title:  "centos7安装部署k8s并进行service外部映射"
categories: 运维
tags: [k8s,service,deployment,centos7]
author: ligson
description: centos7安装部署k8s并进行service外部映射
---

1. 目标
    1. 搭建一个master和两个node
    2. ip地址
        ```bash
        master: 192.168.1.4
        node1: 192.168.1.7
        node2: 192.168.1.8
        ```
    3. 创建一个service 30080可以对外访问
2. 基础软件安装
    1. master

        ```bash
        yum install -y etcd kubernetes-master ntp flannel
        ```
    2. node

        ```bash
        yum install -y etcd kubernetes-node ntp flannel docker
        ```
3. 做成系统服务

    1. master
        ```bash
        systemctl enable etcd kube-apiserver     kube-controller-manager kube-scheduler
        ```
    2. node
        ```bash
        systemctl enable etcd flanneld kube-proxy kubelet docker
        ```
4. 开始前准备

    1. 在master和node安装工具软件(最好yum update -y)
        ```bash
        yum install -y wget elinks ntp gcc net-tools telnet
        systemctl enable ntpd
        # 启动服务
        systemctl start ntpd
        # 设置亚洲时区
        timedatectl set-timezone Asia/Shanghai
        # 启用NTP同步
        timedatectl set-ntp yes
        # 重启ntp服务
        systemctl restart ntpd
        systemctl stop firewalld.service
        systemctl disable firewalld.service
        init 6
        ```
    2.在每个node上执行

    ```bash
        docker pull registry.access.redhat.com/rhel7/pod-infrastructure:latest

        #如果提示下面错误:

        details: (open /etc/docker/certs.d/registry.access.redhat.com/redhat-ca.crt: no such file or directory)

        #执行下面命令
        cd

        wget http://mirror.centos.org/centos/7/os/x86_64/Packages/python-rhsm-certificates-1.19.10-1.el7_4.x86_64.rpm

        rpm2cpio python-rhsm-certificates-1.19.10-1.el7_4.x86_64.rpm | cpio -iv --to-stdout ./etc/rhsm/ca/redhat-uep.pem | tee /etc/rhsm/ca/redhat-uep.pem
    ```
5. 配置文件

    1. master

        1. etcd服务启动文件: /usr/lib/systemd/system/etcd.service

        ```bash
            [Unit]
            Description=Etcd Server
            After=network.target
            After=network-online.target
            Wants=network-online.target

            [Service]
            Type=notify
            WorkingDirectory=/var/lib/etcd/
            EnvironmentFile=-/etc/etcd/etcd.conf
            User=etcd
            # set GOMAXPROCS to number of processors
            ExecStart=/bin/bash -c "GOMAXPROCS=$(nproc) /usr/bin/etcd\
                    --name=\"${ETCD_NAME}\"\
                    --data-dir=\"${ETCD_DATA_DIR}\"\
                    --listen-client-urls=\"${ETCD_LISTEN_CLIENT_URLS}\"\
                    --listen-peer-urls=\"${ETCD_LISTEN_PEER_URLS}\"\
                    --advertise-client-urls=\"${ETCD_ADVERTISE_CLIENT_URLS}\"\
                    --initial-cluster-token=\"${ETCD_INITIAL_CLUSTER_TOKEN}\"\
                    --initial-cluster=\"${ETCD_INITIAL_CLUSTER}\"\
                    --initial-cluster-state=\"${ETCD_INITIAL_CLUSTER_STATE}\""
            Restart=on-failure
            LimitNOFILE=65536
        ```

        2. etcd配置文件(cat /etc/etcd/etcd.conf |grep -v '^#'):

        ```bash
            ETCD_DATA_DIR="/var/lib/etcd/etcd1.etcd"
            ETCD_LISTEN_PEER_URLS="http://192.168.1.4:2380"
            ETCD_LISTEN_CLIENT_URLS="http://192.168.1.4:2379,http://127.0.0.1:2379"
            ETCD_NAME="etcd1"
            ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.1.4:2380"
            ETCD_ADVERTISE_CLIENT_URLS="http://192.168.1.4:2379"
            ETCD_INITIAL_CLUSTER="etcd1=http://192.168.1.4:2380,etcd2=http://192.168.1.7:2380,etcd3=http://192.168.1.8:2380"
            ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
            ETCD_INITIAL_CLUSTER_STATE="new"
        ```

        3. k8s主配置文件( cat /etc/kubernetes/config |grep -v '^#'):

        ```bash
            KUBE_LOGTOSTDERR="--logtostderr=true"
            KUBE_LOG_LEVEL="--v=0"
            KUBE_ALLOW_PRIV="--allow-privileged=false"
            KUBE_MASTER="--master=http://192.168.1.4:8080"
        ```

        4. k8s apiserver(cat /etc/kubernetes/apiserver |grep -v '^#')

        ```bash

            KUBE_API_ADDRESS="--insecure-bind-address=0.0.0.0"

            KUBE_ETCD_SERVERS="--etcd-servers=http://192.168.1.4:2379"

            KUBE_SERVICE_ADDRESSES="--service-cluster-ip-range=10.168.0.0/16"

            KUBE_ADMISSION_CONTROL="--admission-control=AlwaysAdmit"

            KUBE_API_ARGS=""
        ```

        5. k8s 控制器文件(cat /etc/kubernetes/controller-manager |grep -v '^#')

        ```bash
        KUBE_CONTROLLER_MANAGER_ARGS=""
        ```

        6. scheduler(cat /etc/kubernetes/scheduler |grep -v '^#')

        ```bash
        KUBE_SCHEDULER_ARGS=""
        ```
    2. node 配置文件

        1. etcd服务启动文件: /usr/lib/systemd/system/etcd.service

            **同master配置文件，master、node1 、node2 都一样**

        2. etcd配置文件(cat /etc/etcd/etcd.conf |grep -v '^#'):
            ```bash
                #node1配置文件
                ETCD_DATA_DIR="/var/lib/etcd/etcd2.etcd"
                ETCD_LISTEN_PEER_URLS="http://192.168.1.7:2380"
                ETCD_LISTEN_CLIENT_URLS="http://192.168.1.7:2379,http://127.0.0.1:2379"
                ETCD_NAME="etcd2"
                ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.1.7:2380"
                ETCD_ADVERTISE_CLIENT_URLS="http://192.168.1.7:2379,http://127.0.0.1:2379"
                ETCD_INITIAL_CLUSTER="etcd1=http://192.168.1.4:2380,etcd2=http://192.168.1.7:2380,etcd3=http://192.168.1.8:2380"
                ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
                ETCD_INITIAL_CLUSTER_STATE="new"

                #node2配置文件
                ETCD_DATA_DIR="/var/lib/etcd/etcd3.etcd"
                ETCD_LISTEN_PEER_URLS="http://192.168.1.8:2380"
                ETCD_LISTEN_CLIENT_URLS="http://192.168.1.8:2379,http://127.0.0.1:2379"
                ETCD_NAME="etcd3"
                ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.1.8:2380"
                ETCD_ADVERTISE_CLIENT_URLS="http://192.168.1.8:2379,http://127.0.0.1:2379"
                ETCD_INITIAL_CLUSTER="etcd1=http://192.168.1.4:2380,etcd2=http://192.168.1.7:2380,etcd3=http://192.168.1.8:2380"
                ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster"
                ETCD_INITIAL_CLUSTER_STATE="new"
            ```

            **区别：**

                1. ETCD_DATA_DIR和ETCD_NAME名字不一样
                2. ETCD_INITIAL_CLUSTER完全一样，所有etcd节点的ip地址
                3. ip不一样
        3. k8s主配置文件( cat /etc/kubernetes/config |grep -v '^#'):

        ```
            KUBE_LOGTOSTDERR="--logtostderr=true"

            KUBE_LOG_LEVEL="--v=0"

            KUBE_ALLOW_PRIV="--allow-privileged=false"

            KUBE_MASTER="--master=http://192.168.1.4:8080"
        ```

        **node1和node2完全一致**
        4. kublet(cat /etc/kubernetes/kubelet |grep -v '^#')
        ```bash
        KUBELET_ADDRESS="--address=192.168.1.7"
        KUBELET_HOSTNAME="--hostname-override=192.168.1.7"
        KUBELET_API_SERVER="--api-servers=http://192.168.1.4:8080"
        KUBELET_POD_INFRA_CONTAINER="--pod-infra-container-image=registry.access.redhat.com/rhel7/pod-infrastructure:latest"
        KUBELET_ARGS=""
        ```
        **node1和node2的ip不同**
        5. proxy文件(cat /etc/kubernetes/proxy |grep -v '^#')
        ```
        KUBE_PROXY_ARGS="--bind-address=0.0.0.0"
        ```
        **node1和node2完全一致**

6. etcd设置

    1. 在master和node上分别启动etcd服务

        ```bash
        systemctl start etcd
        ```
    2. 检查是否正常启动

        ```bash
        etcdctl cluster-health

        etcdctl member list
        ```

        正常的话会看到如下输出:

        ```bash
        #etcdctl cluster-health输出
        member 6b23ec425a4bbc is healthy: got healthy result from http://127.0.0.1:2379
        member 1667b822635af5d is healthy: got healthy result from http://192.168.1.4:2379
        member 9af728b0ff83aa27 is healthy: got healthy result from http://127.0.0.1:2379
        cluster is healthy

        #etcdctl member list输出
        6b23ec425a4bbc: name=etcd2 peerURLs=http://192.168.1.7:2380 clientURLs=http://127.0.0.1:2379,http://192.168.1.7:2379 isLeader=false
        1667b822635af5d: name=etcd1 peerURLs=http://192.168.1.4:2380 clientURLs=http://192.168.1.4:2379 isLeader=true
        9af728b0ff83aa27: name=etcd3 peerURLs=http://192.168.1.8:2380 clientURLs=http://127.0.0.1:2379,http://192.168.1.8:2379 isLeader=false
        ```
    3. 设置网络

    ```bash
        etcdctl mk /atomic.io/network/config '{ "Network": "10.168.0.0/16" }'
    ```
7. 启动服务
    1. master
    ```bash
    systemctl restart flanneld kube-apiserver kube-controller-manager kube-scheduler
    ```
    2. node
    ```bash
    systemctl restart flanneld kube-proxy kubelet docker
    ```
8. 验证服务

    1. 子网验证

    ```bash
        [root@master ~]# etcdctl ls /atomic.io/network
        /atomic.io/network/config
        /atomic.io/network/subnets
        [root@master ~]# etcdctl get /atomic.io/network/config
        {"Network": "172.168.0.0/16"}
        [root@master ~]# etcdctl ls /atomic.io/network/subnets
        /atomic.io/network/subnets/172.168.62.0-24
        /atomic.io/network/subnets/172.168.53.0-24
        /atomic.io/network/subnets/172.168.51.0-24
        [root@master ~]# etcdctl ls /atomic.io/network/subnets/172.168.62.0-24
        /atomic.io/network/subnets/172.168.62.0-24
        [root@master ~]# etcdctl get /atomic.io/network/subnets/172.168.62.0-24
        {"PublicIP":"192.168.1.7"}
        [root@master ~]# etcdctl get /atomic.io/network/subnets/172.168.53.0-24
        {"PublicIP":"192.168.1.8"}
        [root@master ~]# etcdctl get /atomic.io/network/subnets/172.168.51.0-24
        {"PublicIP":"192.168.1.4"}
        [root@master ~]#
    ```
    2. master验证一下命令
    ```bash
    kubectl get node
    正常输出
    NAME          STATUS    AGE
    192.168.1.7   Ready     21h
    192.168.1.8   Ready     21h
    ```
9. 创建一个deployment(master上执行一下命令)

    1. 建立一个nginx-deploy.yml文件:

    ```yaml
        apiVersion: extensions/v1beta1
        kind: Deployment
        metadata:
          name: nginx
        spec:
          replicas: 2
          selector:
                matchLabels:
                        app: nginx
          template:
            metadata:
              labels:
                app: nginx
            spec:
                containers:
                - name: nginx
                  image: nginx
                  imagePullPolicy: IfNotPresent
                  ports:
                  - name: http
                    containerPort: 80
                    protocol: TCP
    ```

    2. 执行命令: kutectl create -f nginx-deploy.yml

    3. 验证：

    ```bash
        [root@master ~]# kubectl get deployments
        NAME      DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
        nginx     2         2         2            2           1h
        [root@master ~]# kubectl get pods -o wide
        NAME                     READY     STATUS    RESTARTS   AGE       IP             NODE
        nginx-1838792130-dcnbq   1/1       Running   1          1h        172.168.53.2   192.168.1.8
        nginx-1838792130-s9plp   1/1       Running   1          1h        172.168.62.2   192.168.1.7
    ```

10. 创建一个service(master上执行一下命令)

    1. service创建文件nginx-svc.yml

    ```bash
        apiVersion: v1
        kind: Service
        metadata:
          name: nginx
        spec:
          type: NodePort
          #sessionAffinity: ClientIP
          selector:
            app: nginx
          ports:
          - port: 80
            targetPort: 80
            nodePort: 30080
            protocol: TCP
    ```

    2. 执行创建命令kubectl create -f nginx-svc.yml

    3. 验证

    ```bash
        [root@master ~]# kubectl get services
        NAME         CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
        kubernetes   10.168.1.1     <none>        443/TCP        21h
        nginx        10.168.16.34   <nodes>       80:30080/TCP   1h
    ```
11. 验证

    1. 在node1上执行:elinks http://127.0.0.1:30080

    ```bash
        [root@node1 ~]# elinks --dump http://127.0.0.1:30080
                                   Welcome to nginx!

           If you see this page, the nginx web server is successfully installed and
           working. Further configuration is required.

           For online documentation and support please refer to [1]nginx.org.
           Commercial support is available at [2]nginx.com.

           Thank you for using nginx.

        References

           Visible links
           1. http://nginx.org/
           2. http://nginx.com/
    ```

12. 遗留问题

    1. node1和node2各自访问elinks --dump http://127.0.0.1:30080 时好时坏

    2. 如果使用ip 不能相互访问
