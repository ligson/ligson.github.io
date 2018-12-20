---
layout: post
title:  "centos7下使用vsftpd搭建本地局域网yum源"
categories: 运维
tags: [centos7,yum,vsftpd,仓库]
author: ligson
description: centos7安装部署k8s并进行service外部映射
---
 
    场景:
    公司有很多开发服务器，但是为了安全不能让所有的都上网，只能其中的几台可以上网，但是不能上网就不能通过yum安装软件，感觉很不方面，有其中的一台可以上网的服务器做成yum仓库源，这样其他服务器可以通过这个yum安装软件，并且速度也快
    
1. 服务端搭建

    ```bash
        #查看现有仓库列表
        yum repolist
        #下载阿里云yum源配置文件
        wget -O /etc/yum.repos.d/epel.repo https://mirrors.aliyun.com/repo/epel-7.repo
        #查看
        yum repolist
        #安装仓库源配置工具
        yum install yum-utils -y
    ```

2.建立仓库源
    
    ```bash
        mkdir /share/yumrepos
        #同步仓库比较慢12G空间左右
        reposync -r epel -p /share/yumrepos/
        #创建仓库缓存
        createrepo -pdo /share/yumrepos/epel/ /share/yumrepos/epel/
     ```

3. 安装vsftpd  

    ```bash
        yum install vsftpd
        systemctl enable  vsftpd.service
        systemctl restart  vsftpd.service
        firewall-cmd --permanent --add-service=ftp
        firewall-cmd --reload
        #可以关闭防火墙
        systemctl stop firewalld.service
        systemctl disable firewalld.service
        #禁用selinux
        #1. 永久有效
            修改 /etc/selinux/config 文件中的 SELINUX="" 为 disabled ，然后重启。
        #2. 即时生效
            setenforce 0
    ```

4.  配置vsftpd

    ```
        cat /etc/vsftpd/vsftpd.conf |grep -v ^#
        anonymous_enable=YES
        local_enable=YES
        write_enable=NO
        local_umask=022
        anon_upload_enable=NO
        anon_mkdir_write_enable=NO
        dirmessage_enable=YES
        xferlog_enable=YES
        connect_from_port_20=YES
        xferlog_file=/var/log/xferlog
        xferlog_std_format=YES
        chroot_local_user=NO
        chroot_list_enable=NO
        ls_recurse_enable=YES
        listen=YES
        listen_ipv6=NO
        
        tcp_wrappers=YES
        anon_root=/share
        no_anon_password=YES
        ftp_username=ftp
    ```

5. 客户端配置

    ```
    #/etc/yum.repos.d/ 下面其他的仓库可以先移走，毕竟客户端上不了网也没有用处
    cat /etc/yum.repos.d/CentOS-Media.repo
        [c7-media]
        name=CentOS-$releasever - Media
        baseurl=ftp://10.10.65.61/yumrepos/epel/
        gpgcheck=0
        enabled=1
        gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

    yum clean all
    yum makecache
    ```
