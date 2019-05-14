# gitlab 容器化安装

#### 准备工作
gitlab/gitlab-ce容器需要先创建三个目录,分别存放应用数据、日志和配置文件:
```bash
 mkdir -p /gitlab/data
 mkdir -p /gitlab/logs
 mkdir -p /gitlab/config
```

#### 运行gitlab
```bash
docker run -itd -p 443:443 -p 80:80 -p 2222:22 --name gitlab --restart always -v /gitlab/config:/etc/gitlab -v /gitlab/logs:/var/log/gitlab -v /gitlab/data:/var/opt/gitlab gitlab/gitlab-ce:latest
```

#### 配置gitlab：
修改 gitlab 配置，设置 external_url 参数即可：
![gitlab](/images/gitlab-config.png)

#### 访问方式
widows:修改 C:\windows\system32\drivers\etc\hosts文件  
linux修改:/etc/hosts  
```
10.86.77.36   gitlab.abc.com
```

设置下管理员账户的密码：
![gitlab](/images/gitlab.png)
然后使用root和刚设置的密码登录即可：
![gitlab-login](/images/gitlab-login.png)
