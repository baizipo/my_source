#### metadata-server介绍
- [ ] 功能

> 虚拟机启动时候需要注入hostname、password、public-key、network-info之类的信息，以便虚拟机能够被租户管理，metadata-server使用上更为灵活，但是他们都依赖镜像内部必须装有cloud-init组件

- [ ] cloud-init组件介绍
> cloud-init是一个在启动的时候定制你的Iaas平台中虚拟机的包，它可以帮助你重新定义你的虚拟机而不需要重新安装，只需要加入对应的配置项即可。在Ec2中有很多镜像都安装了cloud-init来方便用户定制自己的虚拟机。它可以让你在虚拟机启动的时候设置语言环境，设置主机名，甚至生成私钥，添加用户自己的ssh公钥到虚拟机.ssh/authorized_keys, 设置临时挂载点等等,[cloud-init官方资料](https://cloudinit.readthedocs.io/en/latest/)

- [ ] metadata元数据服务
> metadata字面上是元数据，是一个不容易理解的概念。在除了openstack的其他场合也经常会碰到。openstack里的metadata，是提供一个机制给用户，可以设定每一个instance 的参数。Amazon首先提出了metadata的概念，并搭建了metadata的服务，这个服务的公网IP是169.254.169.254，通常虚拟机通过cloud-init发出的请求是:http://169.254.169.254/latest/meta-data，后来很多人给亚马逊定制了一些操作系统的镜像，比如 ubuntu, fedora, centos 等等，而且将里面获取 metadta 的api地址也写死了。所以opentack为了兼容，保留了这个地址169.254.169.254。然后通过iptables nat映射到真实的api上。

- [ ] nova中的metadata-server
> metadata-server的具体实现是在nova-api组件中，nova.conf中与metadata有关的配置如下:
```
enabled_apis=ec2,osapi_compute,metadata

# OpenStack metadata service manager
metadata_manager=nova.api.manager.MetadataManager

# IP address for metadata api to listen
metadata_listen=0.0.0.0

# port for metadata api to listen
metadata_listen_port=8775

# Number of workers for metadata service
metadata_workers=

# 和neutron-metadata-agent 通信相关
service_neutron_metadata_proxy=True
neutron_metadata_proxy_shared_secret=
```
---

- [ ] 具体的代码实现在 api/metadata 下
```
metadata=metadata {
  __init__.py
  base.py
  handler.py
  password.py
  vendordata_json.py
}
```
- 脚本所在目录
```
/usr/lib/python2.7/dist-packages/nova/api/metadata
```
---
- [ ]   metadata在openstack中的使用

- 虚拟机通过cloud-init组件请求169.254.169.254这个地址的metadata服务，这时这个请求会有两种方式处理
- 先看架构图,虚拟机是用metadata的流程，与neutron的结合使用:

![](https://github.com/baizipo/my_source/tree/master/image/api-agent.jpeg)

> 当虚拟机所在的子网拥有网关而且连接了l3-router，则通过qrouter的namespace中的iptables处理； 当虚拟机所在的子网没有网关，是个封闭的子网，那么dhcp服务的虚拟网卡会添加一个169.254.169.254的ip；接收的cloud-init请求由ns-metadata-proxy处理，ns-metadata-proxy与metadata-agent通过unix domain socket实现IPC，实现将对ns-metadata-proxy的请求交给metadata-agent处理。metadata-agent接收请求，将请求交给metadata-server的真实实现者nova-api。

---

- [ ] 虚拟机所在子网连接了l3-router的处理方式

#### 虚拟机发送请求
- 虚拟机启动时会访问169.254.169.254获取一些内容
```
curl http://169.254.169.254/latest/meta-data
```
- 虚拟机内部没有特殊的路由，所以数据包会直接发送到虚拟机的默认网关，而默认网关是在network node上

---

#### namespace-metadata-proxy
- 因为使用了namespace，在network node上每个namespace里都会有相应的iptables规则和网络设备

- 查看对应的iptables规则

```
$ip netns exec qrouter-82fce223-283e-4349-bb81-53cc52aafe0d iptables -t nat -nL

Chain neutron-l3-agent-PREROUTING (1 references)
target     prot opt source               destination         
REDIRECT   tcp  --  0.0.0.0/0            169.254.169.254      tcp dpt:80 redir ports 8775

$ip netns exec qrouter-82fce223-283e-4349-bb81-53cc52aafe0d iptables  -nL

Chain neutron-l3-agent-INPUT (1 references)
target     prot opt source               destination         
ACCEPT     all  --  0.0.0.0/0            0.0.0.0/0            mark match 0x1/0xffff
DROP       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:8775

```
- iptables规则中，会把目的地址169.254.169.254的数据包，重定向到本地端口8775，那么看一下network node上，在该端口的监听进程
```
$ip netns exec qrouter-82fce223-283e-4349-bb81-53cc52aafe0d netstat -nlpt｜grep 8775

Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:8775            0.0.0.0:*               LISTEN      21687/python2.7

查看该进程
$ps -f --pid 21687 | fold -s
UID        PID  PPID  C STIME TTY          TIME CMD
neutron  21687     1  0 Aug25 ?        00:00:00 /usr/bin/python2.7 
/usr/bin/neutron-ns-metadata-proxy 
--pid_file=/var/lib/neutron/external/pids/82fce223-283e-4349-bb81-53cc52aafe0d.p
id --metadata_proxy_socket=/var/lib/neutron/metadata_proxy 
--router_id=82fce223-283e-4349-bb81-53cc52aafe0d --state_path=/var/lib/neutron 
--metadata_port=8775 --metadata_proxy_user=117 --metadata_proxy_group=124 
--verbose 
--log-file=neutron-ns-metadata-proxy-82fce223-283e-4349-bb81-53cc52aafe0d.log 
--log-dir=/var/log/neutron

```
- 对应代码块
```
$vim  /usr/lib/python2.7/dist-packages/neutron/agent/metadata/namespace_proxy.py
   
    def _proxy_request(self, remote_address, method, path_info,
                       query_string, body):
        headers = {
            'X-Forwarded-For': remote_address,
        }

        if self.router_id:
            headers['X-Neutron-Router-ID'] = self.router_id
        else:
            headers['X-Neutron-Network-ID'] = self.network_id
```
- 可见，启用namespace场景下，对于每一个router，都会创建这样一个进程。该进程监听8775端口，其主要功能：
- 向请求头部添加X-Forwarded-For和X-neutron-Router-ID，分别表示虚拟机的fixedIP和router的ID
- 将请求代理至Unix domain socket（/var/lib/neutron/metadata_proxy）

---

#### neutron Metadata Agent
- network node上的metadata agent监听/var/lib/neutron/metadata_proxy：
```
$netstat -lxp | grep metadata 
unix  2      [ ACC ]     STREAM     LISTENING     933257   24409/python2.7     /var/lib/neutron/metadata_proxy

$ps -f --pid 24409 | fold -s
UID        PID  PPID  C STIME TTY          TIME CMD
neutron  24409     1  0 Aug25 ?        00:01:10 /usr/bin/python2.7 
/usr/bin/neutron-metadata-agent --config-file=/etc/neutron/neutron.conf 
--config-file=/etc/neutron/metadata_agent.ini 
--log-file=/var/log/neutron/metadata-agent.log
```
- 对应代码块
```
$vim /usr/lib/python2.7/dist-packages/neutron/agent/metadata/agent.py

@webob.dec.wsgify(RequestClass=webob.Request)
    def __call__(self, req):
        try:
            LOG.debug("Request: %s", req)

            instance_id, tenant_id = self._get_instance_and_tenant_id(req)
            if instance_id:
                return self._proxy_request(instance_id, tenant_id, req)
            else:
                return webob.exc.HTTPNotFound()

        except Exception:
            LOG.exception(_LE("Unexpected error."))
            msg = _('An unknown error has occurred. '
                    'Please try your request again.')
            explanation = six.text_type(msg)
            return webob.exc.HTTPInternalServerError(explanation=explanation)
```

- 该进程的功能是，根据请求头部的X-Forwarded-For和X-Quantum-Router-ID参数，向Quantum service查询虚拟机ID，然后向Nova Metadata服务发送请求（默认端口8775），消息头：X-Forwarded-For，X-Instance-ID、X-Instance-ID-Signature分别表示虚拟机的fixedIP，虚拟机ID和虚拟机ID的签名。
