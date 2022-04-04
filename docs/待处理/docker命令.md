# docker 常用命令

## 镜像

```bash
docker search  # 搜索镜像
    -f  # 过滤镜像
docker search mysql -f STARS=3000  # 搜索 starts 大于等于 3000 的 mysql 镜像
docker images  # 现在镜像
    -a  # 显示所有
    -q  # 只显示镜像 id
docker pull  # 拉取镜像
docker rmi  # 删除镜像
docker rmi $(docker images -aq)  # 删除所有镜像
```

## 容器

```bash
docker update [OPTIONS] CONTAINER [CONTAINER...]  # 更新容器配置
docker update --restart=always abebf7571666  # 更新容器 abebf7571666 重启配置为总是重启
```

## 创建容器

`docker run -itd --name -e TZ=Asia/Shanghai 镜像名 镜像名:tag`
`-e TZ=Asia/Shanghai`指定时区

## 容器生成镜像

`docker commit -a "h" -m "message" 容器名 镜像名`

## 查看空间占用

docker system df

## 清理

docker system prune
