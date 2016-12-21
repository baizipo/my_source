#Openstack部署文档(all in one)

###一  . Fuel 节点部署
  - 准备Fuel9.0光盘或优盘启动盘

  - 安装Fuel节点
     1.Fuel节点boot from cd/dvd or 从优盘启动安装

     2.选择第一项,如下图,
       ![](https://github.com/baizipo/my_source/blob/master/image/Fuel/fuel.png)
 
  - 配置Fuel参数,需要更改的几项如下:

　  ![](https://github.com/baizipo/my_source/blob/master/image/Fuel/fuel1.png) 
    
       1. Fuel User  :配置Fuel用户密码,保持默认即可
       2. Network Setup :配置pxe网络接口和pxe网络使用地址
       3. Security Setup :配置允许访问访问Fuel节点的网段
       4. Pxe Setup :配置pxe网络的dhcp池
       5. Bootstrap Image :这里选择Skip building bootstrap image,我们用本地模板
       6. Feature groups :这里选择Advanced Features
       7. Quit Setup : 选择Save and Quit

  - 部署local mirrors 和bootstrap
 
       1.cp 做好的local mirrors(fuel9-mirrors.tar.xz)到Fuel节点的/var/www/nailgun目录下
       2.解压local mirrors文件,制作local mirrors
        
          #tar -Jxf  fuel9-mirrors.tar.xz

          #fuel-createmirror

      3.修改编辑配置文件,修改如下字段,修改uri地址为设置的pxe网络使用的地址
```
         ＃vim /etc/fuel-bootstrap-cli/fuel_bootstrap_cli.yaml
　　
         repos:
  　　  - name: ubuntu
      　　section: "main universe multiverse"
      　　uri: "http://10.10.20.2:8080/mirrors/ubuntu"
     　　 priority:
    　　  suite: trusty
     　　 type: deb
   　　 - name: ubuntu-updates
    　　  section: "main universe multiverse"
      　　uri: "http://10.10.20.2:8080/mirrors/ubuntu"
      　　priority:
     　　 suite: trusty-updates
      　　type: deb
  　　  - name: ubuntu-security
      　　section: "main universe multiverse"
     　　 uri: "http://10.10.20.2:8080/mirrors/ubuntu"
      　　priority:
     　　 suite: trusty-security
     　　 type: deb
    　　- name: mos
     　　 section: "main restricted"
     　　 uri: "http://127.0.0.1:8080/ubuntu/x86_64"
      　　priority: 1050
     　　 suite: mos9.0
     　　 type: deb
   　　 - name: mos-updates
      　　section: "main restricted"
      　　uri: "http://10.10.20.2:8080/mirrors/mos-repos/ubuntu/9.0"
      　　priority: 1050
     　　 suite: mos9.0-updates
      　　type: deb
    　　- name: mos-security
      　　section: "main restricted"
     　　 uri: "http://10.10.20.2:8080/mirrors/mos-repos/ubuntu/9.0"
      　　priority: 1050
      　　suite: mos9.0-security
     　　 type: deb
 　　   - name: mos-holdback
      　　section: "main restricted"
      　　uri: "http://10.10.20.2:8080/mirrors/mos-repos/ubuntu/9.0"
      　　priority: 1100
      　　suite: mos9.0-holdback
      　　type: deb
  　　skip_default_img_build: true
 　　direct_repo_addresses:
   　　　- "192.168.31.253"
    　　　- "127.0.0.1" 　
```

      4.制作bootstrap
```
      #fuel-bootstrap build  需要等5-10分钟,会在tmp目录下生成一个tgr.gz的压缩文件

      #fuel-bootstrap import  /tmp/xxx.tgr.gz  导入生成的tar.gz文件

      #fuel-bootstrap activate xxx    激活文件

```

      5.登陆Fuel　webui

       https://10.10.20.2:8443/#login     用户名密码:   admin/admin

      6.创建openstack环境

       登陆后-新建openstack环境-输入名称(openstack)-计算(qemu-kvm)-网络设置(Neutron 并使用 VLAN)-
      后端存储(根据实际情况选择ceph或者是kvm)－ 附加服务(安装 Ceilometer（OpenStack 计费）)－完成－新建

      7.配置openstack环境参数
 
    网络栏-根据用户提供的网络地址参数进行相应设置,参考openstack环境部署需求ppt，配置如下图:
![](https://github.com/baizipo/my_source/blob/master/image/Fuel/NETWORK.png) 
其他里面设置,NTP server list服务器地址为Fuel节点地址(pxe地址),如下图:
![](https://github.com/baizipo/my_source/blob/master/image/Fuel/NTP.png) 
      设置栏:计算配置-Hypervisor type选择kvm，根据情况选择是否勾选Nova quotas,如下图:
![](https://github.com/baizipo/my_source/blob/master/image/Fuel/计算.png) 

 存储配置-根据实际适用情况选择是使用ceph还是lvm，如果是ceph的话Ceph object replication factor在这定义副本数,如下图:
 ![](https://github.com/baizipo/my_source/blob/master/image/Fuel/storage.png) 
   
      8.修改虚拟机模板文件/etc/puppet/modules/osnailyfacter/templates/vm_libvirt.erb将br-mesh修改为br-prv，并且将type设置为openvswitch

```
      #vim /etc/puppet/modules/osnailyfacter/templates/vm_libvirt.erb

       <interface type='bridge'>
       <source bridge='br-prv'/>
       <virtualport type='openvswitch'/>
       <model type='virtio'/>
       </interface>
```

      9.baremetal服务器从pxe启动服务器,等Fuel节点发现

     10.在FUEL界面将发现的baremetal节点配置为virt和compute节点，如下图:
![](https://github.com/baizipo/my_source/blob/master/image/Fuel/virtual.png) 
     
     11.在Fuel cli界面创建虚拟机规格(控制器使用),这台作为控制节点的虚拟机会部署在这台物理节点上

```
        #fuel2 node create-vms-conf 2 --conf '{"id":1,"mem":4,"cpu":8}'

```
    12.部署这台虚拟机,可以在FUEL控制台界面点Provision VMs,或者执行命令

```
        #fuel2 env spawn-vms 1
　　
　　　后面的1 为env id
　　
　　**在物理机器安装完后需要手工安装libxslt1.1_1.1.26-8ubuntu1.3_amd64.deb和xmlstarlet_1.5.0-1_amd64.deb这２个包,否则会报依赖性错误然后退出**

      部署完成后会在这台baremetal节点上生成一台虚拟机,该虚拟机配置是上面node create指定的配置
      等Fuel发现找个节点后，配置这个节点，编辑磁盘和接口配置，这虚拟机对应５块网卡,每块网卡对应一个桥配置并应用
```

    13.修改这台虚拟机的网卡配置文件（去掉该虚拟机网卡的vlan tag）

```
        13.1.下载配置文件

         #fuel deployment --env 4 --default

          Default deployment info was downloaded to /etc/fuel/deployment_4

-----------------------------------------------------------------

        #fuel env 查看集群编号

       #fuel --env 4 deployment --node 9 --default　　下载指定node的配置文件

       #fuel --env 4 env --force delete 强制删除


       13.2.修改配置文件

       #vim /etc/fuel/deployment_4/9.yaml

         - action: add-port
         bridge: br-mgmt
         name: eth1.1301
         - action: add-port
         bridge: br-storage
         name: eth1.1302

       修改为

        - action: add-port
        bridge: br-mgmt
        name: eth1
        - action: add-port
       bridge: br-storage
       name: eth1


     13.3.上传修改过的配置文件

　 #fuel deployment --upload --env 4

```
  
### 二 . Openstack部署
  - openstack集群服务器物理组网(各个接口,网络,vlan事先规划好)

  - openstack集群服务器通过pxe网络启动服务器,直到fuel界面出现可选node

  - 配置Fuel webui

  1.添加节点,指定服务器角色(Controller,Compute,Ceph OSD),点击应用,如下图
  ![](https://github.com/baizipo/my_source/blob/master/image/Fuel/node-list.png) 

  2.配置接口,指定pxe,公开,管理,存储,私有网络接口,如下图:
  ![](https://github.com/baizipo/my_source/blob/master/image/Fuel/接口.png) 


  3.配置磁盘,指定各个空间使用大小,ceph根据磁盘性能指定osd盘,journal盘,如下图:
 ![](https://github.com/baizipo/my_source/blob/master/image/Fuel/磁盘.png) 

  - 部署openstack
   修改完以上步骤后,点击环境中控制台界面的Deploy,开始部署openstack

  - 部署完成
   部署完成后,Fuel界面可以直接跳转openstack登陆,如下图,默认用户名密码为admin/admin，点击下图中的Horzon即可登陆openstack环境
  ![](https://github.com/baizipo/my_source/blob/master/image/Fuel/successful.png) 

  - 迁移FUEL master环境

   将FUEL master迁移到node-11这台计算节点上

```
    #fuel-migrate node-11
```

### 三 . 部署后更改

- Fuel 节点安装ansible

```
#yum -y install ansible
```

- cp fuel_extend-master目录里的ansible

```
#cd fuel_extend-master
#cp -r ansible/* /etc/ansible
#chmod +x /etc/ansible/hosts
#vim /etc/ansible/fuel.ini 修改cluster为实际的`fuel env`
cluster = 1

```

- 获取cluster环境角色和节点名称
```
#ansible all --list
#cat /tmp/ansible-fuel.index
```

- 执行命令，如果有节点新加的话,ansible命令还需要更改

```
# 配置localmirror 和 bootstrap
#ansible-playbook /etc/ansible/general.yml -t fuel

#ansible-playbook /etc/ansible/general.yml -e @/etc/ansible/vars.yml
```

- 使用openstack-config修改部分参数，该命令执行的参数会记录在fuel环境中,新加节点不用重新执行

```
#vim fuel_extend-master/openstack-config/openstack.yaml
修改下面value段的值，可参考keystone.conf中memcache_servers的值
cache/memcache_servers:
value: 192.168.33.5:11211,192.168.33.7:11211,192.168.33.8:11211
```
- 执行fuel openstack-config 修改配置文件
```
#fuel openstack-config --env 1 upload --file openstack.yaml
#fuel openstack-config --env 1 --execute --config-id 1
```
