# Docker笔记

## Docker容器终端无法输入中文字符

### 原因

通过`locale`查看当前默认的语言环境为`POSIX`，`POSIX`不支持中文；通过`locale -a` 查看当前容器支持的所有语言环境，将当前语言环境修改为支持中文的语言环境`C.UTF-8`。

```bash
# 进入容器
➜  ~ docker exec -it mysql bash
# 查看容器当前语言环境
root@bee799aeba58:/# locale
LANG=
LANGUAGE=
LC_CTYPE="POSIX"
LC_NUMERIC="POSIX"
LC_TIME="POSIX"
LC_COLLATE="POSIX"
LC_MONETARY="POSIX"
LC_MESSAGES="POSIX"
LC_PAPER="POSIX"
LC_NAME="POSIX"
LC_ADDRESS="POSIX"
LC_TELEPHONE="POSIX"
LC_MEASUREMENT="POSIX"
LC_IDENTIFICATION="POSIX"
LC_ALL=
# 查看容器当前支持的语言环境
root@bee799aeba58:/# locale -a
C
C.UTF-8
POSIX
root@bee799aeba58:/#
```

### 解决方法

1. 临时解决：在进入容器的时候定义`LANG`。

```bash
docker exec -it mysql env LANG=C.UTF-8 bash
```

2. 在启动容器的时候设置`LANG`。

Docker: 

```bash
docker run --name mysql --restart always -p 3306:3306 -e LANG=C.UTF-8 -e MYSQL_ROOT_PASSWORD=123456 -d mysql
```

docker-compose: 

```yaml
version: '3'

services:
  mysql:
    image: mysql
    environment:
      - MYSQL_ROOT_PASSWORD=123456
      - LANG=C.UTF-8
    volumes:
      - mysql-data:/var/lib/mysql
    ports:
      - 3306:3306
    networks:
      community:
    restart: always
    
volumes:
  mysql-data:

networks:
  community:
```

3. 在制作镜像的时候，在Dockerfile中定义`LANG`。
