1. 开启portmap和nfs服务      

   ```shell
   service portmap start      
   
   service nfs start
   ```

2. 将要共享的目录写到exports文件中 假设共享的目录为 /sharedisk/     

   ```
   vim /etc/exports
   
   在exports文件中添加
   
   /sharedisk    192.168.0.0(rw,no_root_squash,async)
   
   #表示将/sharedisk这个目录共享给192.168.0.*这些客户机，括号中的参数设置意义为：
   
   ro                          该主机对该共享目录有只读权限
   rw                         该主机对该共享目录有读写权限
   root_squash         客户机用root用户访问该共享文件夹时，将root用户映射成匿名用户
   no_root_squash   客户机用root访问该共享文件夹时，不映射root用户
   all_squash            客户机上的任何用户访问该共享目录时都映射成匿名用户
   anonuid                将客户机上的用户映射成指定的本地用户ID的用户
   anongid                将客户机上的用户映射成属于指定的本地用户组ID
   sync                      资料同步写入到内存与硬盘中
   async                    资料会先暂存于内存中，而非直接写入硬盘
   insecure                允许从这台机器过来的非授权访问
   ```

3. 重启nfs 或者使用exportfs命令使设置生效      

   ````
    重启nfs：
   
        service nfs restart
   
        用exportfs
   
        exportfs -rv
   
        #exportfs用法
   
        -a ：全部mount或者unmount /etc/exports中的内容 
        -r ：重新mount /etc/exports中分享出来的目录
        -u ：umount 目录
        -v ：将详细的信息输出到屏幕上
   
   这样nfs的服务器端就设置好了。
   ````

4. 在客户端挂载该目录：      

   ```
   在本地创建挂载的目录
   
   mkdir /sharedisk
   
   mount -t nfs 192.168.0.10:/sharedisk  /sharedisk
   
   #将服务器192.168.0.10上的/sharedisk/ 路径挂载到本地
   
   此时，如果服务器端的防火墙有开着的话，将会提示错误，如：
   
   mount: mount to NFS server '192.168.0.10' failed: System Error: No route to host.
   
   我在挂载的时候就被卡在这里了，主要是对防火墙的设置不太熟悉，在网上找了一些文档按照说明做了下还是不行
   
   后来先读了下iptables的资料，搜索了下nfs所需的服务，结合前面看的设置文档，终于搞定。
   
   
   由于nfs服务需要开启 mountd,nfs,nlockmgr,portmapper,rquotad这5个服务，需要将这5个服务的端口加到iptables里面
   
   而nfs 和 portmapper两个服务是固定端口的，nfs为2049，portmapper为111。其他的3个服务是用的随机端口，那就需要
   
   先把这3个服务的端口设置成固定的。
   ```

5. 查看当前这5个服务的端口并记录下来 用rpcinfo -p 

   这里显示 nfs  2049, portmapper  111, 将剩下的三个服务的端口随便选择一个记录下来

    mountd  976 

   rquotad  966 

   nlockmgr  33993 

6. 将这3个服务的端口设置为固定端口     

    vim  /etc/services      

   在文件的最后一行添加：      

   mountd  976/tcp      

   mountd  976/udp      

   rquotad  966/tcp     

   rquotad  966/udp      

   nlockmgr 33993/tcp      

   nlockmgr 33993/udp      

   保存并退出。 

7. 重启下nfs服务。  

   ```shell
   service nfs restart
   ```

8. 在防火墙中开放这5个端口      

   编辑iptables配置文件       

   vim /etc/sysconfig/iptables      

   添加如下行： 

   ```shell
   -A RH-Firewall-1-INPUT -s 192.168.0.0/24 -m state --state NEW -p tcp --dport 111 -j ACCEPT
   -A RH-Firewall-1-INPUT -s 192.168.0.0/24 -m state --state NEW -p tcp --dport 976 -j ACCEPT
   -A RH-Firewall-1-INPUT -s 192.168.0.0/24 -m state --state NEW -p tcp --dport 2049 -j ACCEPT
   -A RH-Firewall-1-INPUT -s 192.168.0.0/24 -m state --state NEW -p tcp --dport 966 -j ACCEPT
   -A RH-Firewall-1-INPUT -s 192.168.0.0/24 -m state --state NEW -p tcp --dport 33993 -j ACCEPT
   
   -A RH-Firewall-1-INPUT -s 192.168.0.0/24 -m state --state NEW -p udp --dport 111 -j ACCEPT
   -A RH-Firewall-1-INPUT -s 192.168.0.0/24 -m state --state NEW -p udp --dport 976 -j ACCEPT
   -A RH-Firewall-1-INPUT -s 192.168.0.0/24 -m state --state NEW -p udp --dport 2049 -j ACCEPT
   -A RH-Firewall-1-INPUT -s 192.168.0.0/24 -m state --state NEW -p udp --dport 966 -j ACCEPT
   -A RH-Firewall-1-INPUT -s 192.168.0.0/24 -m state --state NEW -p udp --dport 33993 -j ACCEPT
   
   ```



    保存退出并重启iptables service iptables restart 重新执行步骤4挂载即可