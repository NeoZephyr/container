## `FROM`
`FROM` 指定一个已经存在的镜像，后续指令基于该镜像进行
```sh
FROM scratch
FROM centos
FROM busybox:latest
```

## `LABEL`
`LABEL` 用于为 Docker 镜像添加元数据
```sh
LABEL maintainer="painpage400@gmail.com"
LABEL version="1.0"
LABEL location="New York" app="Web"
```

## `ENV`
`ENV` 指令用来在镜像构建过程中设置环境变量，增加可维护性
```sh
ENV MYSQL_VERSION="5.6" \
    WEB_ROOT="/data/web/html/"

RUN yum install -y mysql-server="${MYSQL_VERSION:-5.6}"
```

## `RUN`
运行命令并创建新的镜像层，为避免无用分层将多条命令合为一行
```sh
RUN yum update && \
    yum install -y vim && \
    yum clean all && \
    rm -rf /var/cache/yum/*
```

## `ADD`
`ADD` 用来将构建环境下的文件和目录复制到镜像中，如果复制压缩文件会进行解压
```sh
ADD hello /
```

## `COPY`
`COPY` 类似于 `ADD`，但不会进行解压文件，性能优于 `ADD`。如果要添加远程文件/目录，可以使用 `curl` 或者 `wget`
```sh
COPY index.html ${WEB_ROOT:-/data/web/html/}
```

## `WORKDIR`
改变工作目录，若没有目录则创建，尽量使用绝对路径
```sh
WORKDIR /root
```

## `VOLUME`
向基于镜像创建的容器添加卷
```sh
# 为容器创建名为 /data, /home 的挂载点
VOLUME [ "/data", "/home" ]

VOLUME "/data/mysql/"
```

## `EXPOSE`
暴露端口
```sh
EXPOSE 80/tcp
```

## `CMD`
1. 设置容器启动后默认执行的命令和参数
2. 若通过 `docker run` 指定了其它命令，CMD 命令被忽略
3. 若定义了多个 CMD，只有最后一个会执行

```sh
# exec 格式
CMD [ "/bin/bash" ]
CMD [ "/bin/echo", "hello docker" ]
CMD [ "/bin/bash", "-c", "/bin/app", "-p ${port}" ]

# shell 格式
CMD /bin/app -p ${PORT}
CMD echo "hello docker"
```

## `ENTRYPOINT`
让容器以应用程序或者服务的形式运行。该命令不会被忽略，一定会执行
```sh
# exec 格式
ENTRYPOINT ["/bin/echo", "hello docker"]
ENTRYPOINT ["/bin/bash", "-c", "echo hello docker" ]

# shell 格式
ENTRYPOINT echo "hello docker"
ENTRYPOINT /bin/app -p ${PORT}
```

最佳实践
```sh
FROM nginx
ENV NGX_DOC_ROOT="/data/web/html/"

COPY entrypoint.sh /usr/local/bin/
ENTRYPOINT ["entrypoint.sh"]
CMD ["/usr/sbin/nginx", "-g", "daemon off;"]
```
```sh
cat > /etc/nginx/conf.d/www.conf << EOF
server {
    server_name ${HOSTNAME};
    listen ${IP:-0.0.0.0}:${PORT:-80};
    root ${NGX_DOC_ROOT:-/usr/share/nginx/html};
}
EOF

# 替换 cmd 执行
exec "$@"
```

## 构建镜像
```sh
docker build -t <username>/<image name>:<tag> .
```