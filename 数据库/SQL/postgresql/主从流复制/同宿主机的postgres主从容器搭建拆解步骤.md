### 同宿主机的postgres主从容器搭建拆解步骤

* 原生的流复制，slave不能自动升级为master节点，slave节点只能读，导致master节点挂掉之后，整个系统只能查询，不能编辑应用

##### 使用 registry.cn-hangzhou.aliyuncs.com/hegp/postgres-zhparser:11.10-alpine 这个镜像，这个镜像是 zhparser 全文检索官方制作的镜像，被我放到阿里云做加速器的
```
docker pull registry.cn-hangzhou.aliyuncs.com/hegp/postgres-zhparser:11.10-alpine
docker tag registry.cn-hangzhou.aliyuncs.com/hegp/postgres-zhparser:11.10-alpine postgres:11.10-alpine
docker rmi registry.cn-hangzhou.aliyuncs.com/hegp/postgres-zhparser:11.10-alpine

```

#### 创建宿主机网段，并创建目录(用root账号操作，因为操作的那些目录是root路径下的)
```
docker stop pg-master pg-slave
docker rm pg-master pg-slave
docker network rm postgres-network
docker network create --subnet=172.77.0.0/24 postgres-network
rm -rf /opt/soft/postgres/master-slave && mkdir -p /opt/soft/postgres/master-slave

mkdir -p /opt/soft/postgres/master-slave/master
mkdir -p /opt/soft/postgres/master-slave/slave-tmp  # 临时使用的目录，先把主节点的数据复制到这个目录，然后重命名为 slave

```

#### 步骤01) 创建主节点容器
```
docker run -itd --network postgres-network --ip 172.77.0.101 --name pg-master -e "POSTGRES_USER=postgres" -e "POSTGRES_PASSWORD=postgres" -v /opt/soft/postgres/master-slave/master:/var/lib/postgresql/data postgres:11.10-alpine

sleep 15 # 等待postgresql完全跑起来
# 创建主从流复制专用账号
docker exec -it pg-master psql -U postgres -c "CREATE ROLE repuser WITH LOGIN REPLICATION CONNECTION LIMIT 5 PASSWORD 'Q1w2E#';"

```


#### 步骤02) 配置postgresql主节点的参数
```
echo "host replication repuser 172.77.0.102/24 md5" >> /opt/soft/postgres/master-slave/master/pg_hba.conf

cd /opt/soft/postgres/master-slave/master
sed -ri "s|^#?archive_mode\s+.*|archive_mode = on|" postgresql.conf
sed -ri "s|^#?archive_command\s+.*|archive_command = '/bin/date'|" postgresql.conf   # 用该命令来归档logfile segment，这里取消归档。
sed -ri "s|^#?wal_level\s+.*|wal_level = replica|" postgresql.conf   # 开启热备
sed -ri "s|^#?max_wal_senders\s+.*|max_wal_senders = 10|" postgresql.conf
sed -ri "s|^#?wal_keep_segments\s+.*|wal_keep_segments = 16|" postgresql.conf
sed -ri "s|^#?wal_sender_timeout\s+.*|wal_sender_timeout = 60s|" postgresql.conf

# 重启主库使配置生效，使用 pg_ctl stop 安全停止数据库， docker exec -it -u postgres pg-master pg_ctl stop
docker restart pg-master

```

#### 步骤03) 创建从服务器
```
docker run -itd --network postgres-network --ip 172.77.0.102 --name pg-slave -e "POSTGRES_USER=postgres" -e "POSTGRES_PASSWORD=postgres" -v /opt/soft/postgres/master-slave/slave-tmp:/slave-data postgres:11.10-alpine
sleep 10 # 等待从库完成跑起来
# 进入从库容器，同步初始主库数据到 repl 目录
# pg_basebackup -F p --progress -D /slave-data -h 172.77.0.101 -p 5432 -U repuser --password
docker exec -it pg-slave sh -c "pg_basebackup -R -D /slave-data -Fp -Xs -v -P -h 172.77.0.101 -p 5432 -U repuser -W"

```


#### 步骤04) 删掉pg-slave容器，然后把 /opt/soft/postgres/master-slave/slave-tmp 重名为 /opt/soft/postgres/master-slave/slave，然后修改配置参数
```
docker stop pg-slave
docker rm pg-slave
mv /opt/soft/postgres/master-slave/slave-tmp /opt/soft/postgres/master-slave/slave


# /opt/soft/postgres/master-slave/slave/recovery.conf 文件确保有下面的内容 primary_conninfo
cat /opt/soft/postgres/master-slave/slave/recovery.conf
# standby_mode = on    # 指明从库身份
# primary_conninfo = 'user=repuser password=''Q1w2E#'' host=172.77.0.101 port=5432 sslmode=prefer sslcompression=0 krbsrvname=postgres target_session_attrs=any'

echo "recovery_target_timeline = 'latest'" >> /opt/soft/postgres/master-slave/slave/recovery.conf


# /opt/soft/postgres/master-slave/slave/postgresql.conf文件中以下几个参数，并调整如下，该文件在 pg-11.10版本有将近700行，小心改错参数
cd /opt/soft/postgres/master-slave/slave
sed -ri "s|^#?wal_level\s+.*|wal_level = replica|" postgresql.conf  # 开启热备
sed -ri "s|^#?hot_standby\s+.*|hot_standby = on|" postgresql.conf
sed -ri "s|^#?hot_standby_feedback\s+.*|hot_standby_feedback = on|" postgresql.conf

```

#### 重新运行pg-slave容器，然后在master创建数据库，slave节点会同步过来
```
docker run -itd --network postgres-network --ip 172.77.0.102 --name pg-slave -e "POSTGRES_USER=postgres" -e "POSTGRES_PASSWORD=postgres" -v /opt/soft/postgres/master-slave/slave:/var/lib/postgresql/data postgres:11.10-alpine

## 在master创建数据库
docker exec -it pg-master psql -U postgres -c "CREATE DATABASE dbname;"

######################### 停止copy ############################

## 在slave查询所有数据库
docker exec -it pg-slave psql -U postgres -c "select datname from pg_database;";

```