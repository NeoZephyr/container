/etc/docker/daemon.json
```
{
    "registry-mirrors": [
        "http://f1361db2.m.daocloud.io"
    ],
    "insecure-registries": ["192.168.20.10"]
}
```
```sh
sudo systemctl restart docker
```
```sh
docker login 192.168.20.10

docker tag nginx:v1 knode01/library/nginx:v1
docker push knode01/library/nginx:v1
docker pull knode01/library/nginx:v1
```

gitlab
```
docker run -d \
  --name gitlab \
  -p 8443:443 \
  -p 9999:80 \
  -p 9998:22 \
  -v $PWD/config:/etc/gitlab \
  -v $PWD/logs:/var/log/gitlab \
  -v $PWD/data:/var/opt/gitlab \
  -v /etc/localtime:/etc/localtime \
  lizhenliang/gitlab-ce-zh:latest
```

jenkins
```
docker run -d --name jenkins -p 80:8080 -p 50000:50000 -u root  \
   -v /opt/jenkins_home:/var/jenkins_home \
   -v /var/run/docker.sock:/var/run/docker.sock   \
   -v /usr/bin/docker:/usr/bin/docker \
   -v /usr/local/apache-maven-3.5.0:/usr/local/maven \
   -v /usr/local/jdk1.8.0_45:/usr/local/jdk \
   -v /etc/localtime:/etc/localtime \
   --name jenkins jenkins/jenkins:lts
```

使用/root/.ssh中私钥访问gitlab

系统管理-->插件管理-->Installed
搜索git/pipeline，点击安装

添加参数化构建
Pipeline脚本
```
#!/usr/bin/env groovy

def registry = "192.168.31.70"
def project = "welcome"
def app_name = "demo"
def image_name = "${registry}/${project}/${app_name}:${Branch}-${BUILD_NUMBER}"
def git_address = "git@192.168.31.62:/home/git/java-demo.git"
def docker_registry_auth = "b37a147e-5217-4359-8372-17fd9a8edfcc"
def git_auth = "b3e33c8b-c7e0-47b9-baee-d7629d71f154"

pipeline {
    agent any
    stages {
        stage('拉取代码'){
            steps {
              checkout([$class: 'GitSCM', branches: [[name: '${Branch}']], userRemoteConfigs: [[credentialsId: "${git_auth}", url: "${git_address}"]]])
            }
        }

        stage('代码编译'){
           steps {
             sh """
                JAVA_HOME=/usr/local/jdk
                PATH=$JAVA_HOME/bin:/usr/local/maven/bin:$PATH
                mvn clean package -Dmaven.test.skip=true
                """ 
           }
        }

        stage('构建镜像'){
           steps {
                withCredentials([usernamePassword(credentialsId: "${docker_registry_auth}", passwordVariable: 'password', usernameVariable: 'username')]) {
                sh """
                  echo '
                    FROM ${registry}/library/tomcat:v1
                    LABEL maitainer lizhenliang
                    RUN rm -rf /usr/local/tomcat/webapps/*
                    ADD target/*.war /usr/local/tomcat/webapps/ROOT.war
                  ' > Dockerfile
                  docker build -t ${image_name} .
                  docker login -u ${username} -p '${password}' ${registry}
                  docker push ${image_name}
                """
                }
           } 
        }

        stage('部署到Docker'){
           steps {
              sh """
              REPOSITORY=${image_name}
              docker rm -f tomcat-java-demo |true
              docker container run -d --name tomcat-java-demo -p 88:8080 ${image_name}
              """
            }
        }
    }
}
```

添加凭据
1、添加拉取git代码凭据，并获取id替换到上面git_auth变量值。
2、添加拉取harbor镜像凭据，并获取id替换到上面docker_registry_auth变量值。