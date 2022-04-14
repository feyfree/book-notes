```yaml
version: "3.5"
services:
  web-fe:
    build: .
    command: python app.py
    ports:
      - target: 5000
        published: 5000
    networks:
      - counter-net
    volumes:
      - type: volume
        source: counter-vol
        target: /code
  redis:
    image: "redis:alpine"
    networks:
      counter-net:

networks:
  counter-net:

volumes:
  counter-vol:

```

# Commands

```text
# 如果目录下面有docker-compose.yml 的文件名的话， 会执行这个docker-compose.yml
$ docker-compose up  &

# 如果目录不是这么命名的用 -f 命令
$ docker-compose -f prod-equus-bass.yml up

# 后台运行 -d 命令
$ docker-compose -f prod-equus-bass.yml up -d

# 停止
$ docker-compose down

# 查看状态
$ docker-compose ps
       Name                      Command               State           Ports         
-------------------------------------------------------------------------------------
counterapp_redis_1    docker-entrypoint.sh redis ...   Up      6379/tcp              
counterapp_web-fe_1   python app.py                    Up      0.0.0.0:5000->5000/tcp




# 查看每个服务上面的进程
$ docker-compose top
counterapp_redis_1
  UID     PID    PPID   C   STIME   TTY     TIME             CMD        
------------------------------------------------------------------------
polkitd   6158   6141   0   11:29   ?     00:00:01   redis-server *:6379

counterapp_web-fe_1
UID    PID    PPID   C   STIME   TTY     TIME                    CMD                
------------------------------------------------------------------------------------
root   6234   6209   0   11:29   ?     00:00:00   python app.py                     
root   6291   6234   0   11:29   ?     00:00:04   /usr/local/bin/python /code/app.py


```



