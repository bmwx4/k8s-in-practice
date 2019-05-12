# nfs
通过使用nfs volume 可以将现有的 NFS（网络文件系统）共享挂载到您的容器中，nfs可以被多个写入者同时写入；
```yaml
# test-nfs.yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-nfs
spec:
  containers:
    - name: test-nfs
      image: bmwx4/kugo:v1.0
      imagePullPolicy: IfNotPresent
      volumeMounts:
       - name: nfs-storage
         mountPath: /home/nfs-client/
  volumes:
  - name: nfs-storage
    nfs:
     server: 192.168.10.232
     path: "/data/nfs"
```
创建pod并验证:
```bash
# kubectl create -f test-nfs.yaml
# kubectl  exec test-nfs -- df -h
Filesystem                              Size  Used Avail Use% Mounted on
192.168.10.232:/data/nfs                 37G   28G  9.9G  74% /home/nfs-client
```

***前提：***
你需要具备一个NFS 服务，如何配置，见下文：

### NFS 服务配置
添加nfs server配置:
```
# cat /etc/exports
/data/nfs   *(rw,sync,no_root_squash,insecure)
```

启动rpcbind服务,因为启动nfs之前，需要启动rpcbind服务：
```bash
# systemctl start rpcbind.service
A dependency job for rpcbind.service failed. See 'journalctl -xe' for details.
```
可能会报错，查看错误原因:
```bash
# journalctl -xu rpcbind
-- Logs begin at Sat 2019-05-11 15:33:26 CST, end at Sat 2019-05-11 17:06:24 CST. --
May 11 17:06:03 master01 systemd[1]: Dependency failed for RPC bind service.
-- Subject: Unit rpcbind.service has failed
-- Defined-By: systemd
-- Support: http://lists.freedesktop.org/mailman/listinfo/systemd-devel
--
-- Unit rpcbind.service has failed.
--
-- The result is dependency.
May 11 17:06:03 master01 systemd[1]: Job rpcbind.service/start failed with result 'dependency'.
```
那么可能是因为我们把ipv6给禁掉了导致了，可以修改nfs 的systemd配置，把ipv6监听注释掉：
```
[Unit]
Description=RPCbind Server Activation Socket

[Socket]
ListenStream=/var/run/rpcbind.sock

# RPC netconfig can't handle ipv6/ipv4 dual sockets
BindIPv6Only=ipv6-only
ListenStream=0.0.0.0:111
ListenDatagram=0.0.0.0:111
#ListenStream=[::]:111
#ListenDatagram=[::]:111

[Install]
WantedBy=sockets.target
```
再起启动 rpcbind和 nfs service:
```bash
# systemctl start rpcbind.service
# systemctl start nfs
# exportfs
/data/nfs       <world>
```
在其它work node上测试：
```
# mount -t nfs 192.168.10.232:/data/nfs /data
192.168.10.232:/data/nfs on /data type nfs4 (rw,relatime,vers=4.1,rsize=524288,wsize=524288,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=192.168.10.242,local_lock=none,addr=192.168.10.232)
```
