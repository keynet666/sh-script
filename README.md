#!/bin/bash

# 检查是否提供了镜像标签参数
if [ $# -eq 0 ]; then
    echo "错误：请提供镜像标签作为参数"
    echo "用法: $0 <镜像标签>"
    exit 1
fi

TAG=$1
IMAGE_NAME="registry.cn-hongkong.aliyuncs.com/zjl-service-uat/reverse-twitter-blogger:$TAG"
CONTAINER_NAME="reverse-twitter-blogger"

echo "正在拉取最新镜像: $IMAGE_NAME"
# 拉取最新镜像
docker pull $IMAGE_NAME

echo "正在停止并删除现有的容器: $CONTAINER_NAME"
# 停止并删除现有容器（如果存在）
docker stop $CONTAINER_NAME 2>/dev/null
docker rm $CONTAINER_NAME 2>/dev/null

echo "正在创建并启动新容器: $CONTAINER_NAME"
# 创建并启动新容器
docker run -d \
  --name $CONTAINER_NAME \
  -p 8088:8080 \
  $IMAGE_NAME

# 检查容器是否成功启动
if [ $? -eq 0 ]; then
    echo "容器 $CONTAINER_NAME 启动成功"
    
    # 查找并删除旧镜像
    echo "正在查找需要删除的旧镜像..."
    OLD_IMAGES=$(docker images --filter=reference="registry.cn-hongkong.aliyuncs.com/zjl-service-uat/reverse-twitter-blogger:*" --format "{{.ID}}:{{.Tag}}" | grep -v "$TAG" | cut -d':' -f1 | uniq)
    
    if [ ! -z "$OLD_IMAGES" ]; then
        echo "找到以下旧镜像需要删除:"
        echo "$OLD_IMAGES"
        
        # 逐个删除镜像
        for IMAGE_ID in $OLD_IMAGES; do
            echo "正在删除镜像: $IMAGE_ID"
            docker rmi -f $IMAGE_ID 2>/dev/null
            if [ $? -eq 0 ]; then
                echo "成功删除镜像: $IMAGE_ID"
            else
                echo "警告：无法删除镜像 $IMAGE_ID（可能正在被其他容器使用）"
            fi
        done
        echo "旧镜像清理完成"
    else
        echo "没有找到需要删除的旧镜像"
    fi
    
    echo "操作完成"
else
    echo "错误：容器启动失败"
    exit 1
fi
