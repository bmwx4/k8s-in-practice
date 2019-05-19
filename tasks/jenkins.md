# jenkins

### 容器化安装

```bash
$ docker pull jenkins/jenkins:latest
$ mkdir  /home/jenkins  
$ chown -R 1000:1000 /home/jenkins
$ docker run -itd -p 9090:8080 -p 50000:50000 --name jenkins --privileged=true  -v /home/jenkins:/var/jenkins_home jenkins:latest
$ docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

```bash
docker run \
  -u root \
  --rm \
  -d \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins-data:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkinsci/blueocean
```
