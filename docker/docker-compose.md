## docker-compose 命令
```sh
# 生成 image
docker-compose build .

docker-compose images
```
```sh
docker-compose -f docker-compose.yml up

# 以守护进程方式运行
docker-compose -f docker-compose.yml up -d
```
```sh
# 查看服务
docker-compose ps

# 停止服务
docker-compose stop

# 重新启动服务
docker-compose start

# 将已停止的容器删除
docker-compose down

# web 服务扩展
docker-compose up --scale web=3 -d
```
```sh
# 查看服务日志
docker-compose logs
```
```sh
# 进入容器
docker-compose exec mysql bash
```

## docker-compose file
```yaml
version: '3'

services:

  app:
    build:
        context: .
        dockerfile: Dockerfile
    ports:
        - 8080:8080
    links:
        - redis
        - mysql
    depends_on:
        - mysql
        - redis
    environment:
        REDIS_HOST: redis
        DB_HOST: mysql
        DB_PASSWORD: root
    networks:
        - my-bridge

    redis:
        image: redis
        networks: my-bridge

    mysql:
        image: mysql:5.7
        environment:
            MYSQL_ROOT_PASSWORD: root
            MYSQL_DATABASE: app
        volumes:
            - mysql-data:/var/lib/mysql
        networks:
            - my-bridge

volumes:
    mysql-data:

networks:
    my-bridge:
        driver: bridge
```