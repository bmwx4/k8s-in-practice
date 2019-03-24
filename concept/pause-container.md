# Pause容器

Pause容器，又叫Infra容器，本文将探究该容器的作用与原理。

我们知道在kubelet的配置中有这样一个参数：

```bash
KUBELET_POD_INFRA_CONTAINER=--pod-infra-container-image=registry.access.redhat.com/rhel7/pod-infrastructure:latest
```

上面是openshift中的配置参数，kubernetes中默认的配置参数是：

```bash
KUBELET_POD_INFRA_CONTAINER=--pod-infra-container-image=gcr.io/google_containers/pause-amd64:3.0
```

Pause容器，是可以自己来定义，官方使用的`gcr.io/google_containers/pause-amd64:3.0`容器的代码见[Github](https://github.com/kubernetes/kubernetes/tree/master/build/pause)，使用C语言编写。

## Pause容器的作用

我们检查node节点的时候会发现每个node上都运行了很多的pause容器，例如如下。

```bash
$ docker ps
0e031f9223cd        registry.access.redhat.com/rhel7/pod-infrastructure:latest   "/usr/bin/pod"           6 days ago          Up 6 days                                              k8s_POD_nginx-75d67854df-b45mp_default_11a0fc72-4868-11e9-873e-000c29a5444f_0
4b8cea8571be        2d1e5208483c                                                 "httpd-foreground"       6 days ago          Up 6 days                                              k8s_httpod_httpod_default_ea43bf9a-47f3-11e9-873e-000c29a5444f_0
ea848b2fb675        registry.access.redhat.com/rhel7/pod-infrastructure:latest   "/usr/bin/pod"           6 days ago          Up 6 days                                              k8s_POD_httpod_default_ea43bf9a-47f3-11e9-873e-000c29a5444f_0
e2e96aec0af9        httpd                                                        "httpd-foreground"       6 days ago          Up 6 days                                              k8s_httpd_httpd-7db5849b8-9bhxv_default_3f40e762-47f3-11e9-873e-000c29a5444f_0
2c1964b6a062        registry.access.redhat.com/rhel7/pod-infrastructure:latest   "/usr/bin/pod"           6 days ago          Up 6 days                                              k8s_POD_httpd-7db5849b8-9bhxv_default_3f40e762-47f3-11e9-873e-000c29a5444f_0
```

kubernetes中的pause容器主要为每个业务容器提供以下功能：

- 在pod中担任Linux命名空间共享的基础；
- 启用pid命名空间，开启init进程。

在[The Almighty Pause Container](https://www.ianlewis.org/en/almighty-pause-container)这篇文章中做出了详细的说明，pause容器的作用可以从这个例子中看出，首先见下图：

![Pause容器](../images/pause-container.png)



我们首先在节点上运行一个pause容器。

```bash
docker run -d --name pause -p 8880:80 registry.access.redhat.com/rhel7/pod-infrastructure:latest
```

然后再运行一个nginx容器，nginx将为`localhost:2368`创建一个代理。

```bash
$ cat <<EOF >> nginx.conf
error_log stderr;
events { worker_connections  1024; }
http {
    access_log /dev/stdout combined;
    server {
        listen 80 default_server;
        server_name example.com www.example.com;
        location / {
            proxy_pass http://127.0.0.1:2368;
            proxy_set_header  Host $host;
        }
    }
}
EOF
$ docker run -d --name nginx -v `pwd`/nginx.conf:/etc/nginx/nginx.conf --net=container:pause --ipc=container:pause --pid=container:pause nginx
```

然后再为[ghost](https://github.com/TryGhost/Ghost)创建一个应用容器，这是一款博客软件。

```bash
$ docker run -d --name ghost --net=container:pause --ipc=container:pause --pid=container:pause ghost
```

现在访问<http://localhost:8880/>就可以看到ghost博客的界面了。

**解析**

pause容器将内部的80端口映射到宿主机的8880端口，pause容器在宿主机上设置好了网络namespace后，nginx容器加入到该网络namespace中，我们看到nginx容器启动的时候指定了`--net=container:pause`，ghost容器同样加入到了该网络namespace中，这样三个容器就共享了网络，互相之间就可以使用`localhost`直接通信，`--ipc=contianer:pause --pid=container:pause`就是三个容器处于同一个namespace中，init进程为`pause`，这时我们进入到ghost容器中查看进程情况。

```bash
# apt-get update && apt-get install -y procps
# ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0   1024     4 ?        Ss   13:49   0:00 /pause
root         5  0.0  0.1  32432  5736 ?        Ss   13:51   0:00 nginx: master p
systemd+     9  0.0  0.0  32980  3304 ?        S    13:51   0:00 nginx: worker p
node        10  0.3  2.0 1254200 83788 ?       Ssl  13:53   0:03 node current/in
root        79  0.1  0.0   4336   812 pts/0    Ss   14:09   0:00 sh
root        87  0.0  0.0  17500  2080 pts/0    R+   14:10   0:00 ps aux
```

在ghost容器中同时可以看到pause和nginx容器的进程，并且pause容器的PID是1。而在kubernetes中容器的PID=1的进程即为容器本身的业务进程。

## 参考

- [The Almighty Pause Container](https://www.ianlewis.org/en/almighty-pause-container)
- [Kubernetes之Pause容器](https://o-my-chenjian.com/2017/10/17/The-Pause-Container-Of-Kubernetes/)
