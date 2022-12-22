# 背景
由于自建私有云环境采用Ceph作为后端存储，早期算法同学基于CephFS作为分布式共享的文件系统，存储了大量用于训练的数据。
但CephFS由于MDS的性能问题，无法满足直接用于训练读取的诉求（经常会导致CephFS降级），因此需要考虑利用GPU集群自身的Mem和SSD组成分布式的缓存系统来支撑数据处理和训练的场景。

# 选型
通过调研业界最佳实践，基于云原生理念的数据编排+分布式缓存是最近几年的热点，目前控制面面的编排目前仅有Fluid，而数据面的分布式缓存则有多种：Alluxio、JuiceFS、JindoFS等。
进一步分析调研，Alluxio的资历最老，功能强大，但存在容器化能力不足，JuiceFS设计理念直接基于云原生，属于新秀，是后续需要重点对比测试的方向。JindoFS限制较多，暂不展开。

# 架构
![image](https://user-images.githubusercontent.com/22912347/208909421-b3bbdd48-cbfd-4ce8-9ab7-9b6ca5a82cb0.png)


# 实施
基于S3和Ceph对象存储，Fluid官方提供了一些参考资料，具体可以参考附件的链接，
但对接CephFS场景，Alluxio官方文档上说明是可以支持的，但Fluid社区没有提供样例。去github上也没有支持CephFS的Alluxio的容器化版本。

## 1、安装Fluid:
这个比较简单，直接参考Fluid的快速入门指南：
https://github.com/andyzheung/fluid/blob/master/docs/zh/userguide/get_started.md

## 2、制作Alluxio支持CephFS的镜像
Alluxio官网提供了CephFS作为UFS的配置：
https://docs.alluxio.io/os/user/edge/cn/ufs/CephFS.html
需要安装相关的依赖和做软连接。
但是Alluxio官方的Dockerfile在我的环境始终没法正常工作，后发现Fluid社区基于Alluxio的Dockerfile-dev文件制作的Alluxio runtime镜像是可以正常工作的。因此基于此镜像来增量支持CephFS。

### 2.1 使用Alluxio release-2.8.1 分支，修改Dockerfile, 增加ceph.repo源
使用的Dockerfile-dev,增加了：

```sh
#wget cephfs lib
COPY ./ceph.repo /etc/yum.repos.d/ceph.repo 
RUN yum makecache && \
    yum install -y epel-release && \
	yum install -y cephfs-java 
RUN ln -s /usr/lib64/libcephfs_jni.so.1.0.0 /usr/lib64/libcephfs_jni.so && \
    ln -s /usr/lib64/libcephfs.so.2.0.0 /usr/lib64/libcephfs.so && \
    ln -s /usr/share/java/libcephfs.jar ${JAVA_HOME}/jre/lib/ext/libcephfs.jar
#end of cephfs
```
参考个人分支：
https://github.com/andyzheung/alluxio/tree/release-2.8.1/integration/docker

### 2.2 使用Fluid社区的Tool工具构建镜像
使用Fluid社区最新的tag版本：v0.8.2，fork了个人分支：
https://github.com/andyzheung/fluid/tree/v0.8.2/tools/alluxio
暂无改动：
```sh
bash build-image.sh -b release-2.8.1 -c 0433ade -a alluxio-cephfs -f alluxio-fuse-cephfs
```
注意：如果需要push到自己的镜像仓库，请注意修改相应的脚本内容：
主要是build()函数里的内容：
```sh
build()
{
  docker cp ${dev_container_name}:/tmp/alluxio-release-2.8.1-SNAPSHOT-bin.tar.gz /tmp/
  cp /tmp/alluxio-release-2.8.1-SNAPSHOT-bin.tar.gz /alluxio/integration/docker

  cd /alluxio/integration/docker

  GIT_COMMIT=$(git rev-parse --short HEAD)
  echo "GIT_COMMIT=${GIT_COMMIT}"

  # Use aluxio/alluxio-dev image for both Alluxio and Fuse, starting from Alluxio v2.6.2
  # Build Alluxio image
  if test -f /alluxio/integration/docker/Dockerfile-dev; then
    docker build -f Dockerfile-dev -t alluxio/alluxio-dev:release-2.8.1-SNAPSHOT-$GIT_COMMIT --build-arg ALLUXIO_TARBALL=alluxio-release-2.8.1-SNAPSHOT-bin.tar.gz --build-arg ENABLE_DYNAMIC_USER="true" .
  else
    docker build -t alluxio/alluxio-dev:release-2.8.1-SNAPSHOT-$GIT_COMMIT --build-arg ALLUXIO_TARBALL=alluxio-release-2.8.1-SNAPSHOT-bin.tar.gz --build-arg ENABLE_DYNAMIC_USER="true" .
  fi
  # Tag Alluxio image
  docker tag alluxio/alluxio-dev:release-2.8.1-SNAPSHOT-$GIT_COMMIT ${alluxio_image_name}:release-2.8.1-SNAPSHOT-$GIT_COMMIT   #需要根据仓库地址修改

  # Build Fuse image if needed. Tag Fuse image
  if test -f /alluxio/integration/docker/Dockerfile.fuse; then
    docker build -f Dockerfile.fuse -t alluxio/alluxio-fuse:release-2.8.1-SNAPSHOT-$GIT_COMMIT --build-arg ALLUXIO_TARBALL=alluxio-release-2.8.1-SNAPSHOT-bin.tar.gz --build-arg ENABLE_DYNAMIC_USER="true" .
    docker tag alluxio/alluxio-fuse:release-2.8.1-SNAPSHOT-$GIT_COMMIT ${alluxio_fuse_image_name}:release-2.8.1-SNAPSHOT-$GIT_COMMIT  #需要根据仓库地址修改
  else
    docker tag alluxio/alluxio-dev:release-2.8.1-SNAPSHOT-$GIT_COMMIT ${alluxio_fuse_image_name}:release-2.8.1-SNAPSHOT-$GIT_COMMIT   #需要根据仓库地址修改
  fi

  docker push ${alluxio_fuse_image_name}:release-2.8.1-SNAPSHOT-$GIT_COMMIT &  #需要根据仓库地址修改  
  docker push ${alluxio_image_name}:release-2.8.1-SNAPSHOT-$GIT_COMMIT &    #需要根据仓库地址修改
}
```

## 3 更新fuild的helm values.yaml里关于alluxioruntime的镜像

```sh
runtime:
  criticalFusePod: true
  syncRetryDuration: 15s
  mountRoot: /runtime-mnt
  alluxio:
    replicas: 1
    runtimeWorkers: 3
    portRange: 20000-26000
    portAllocatePolicy: bitmap
    enabled: false
    init:
      image: fluidcloudnative/init-users:v0.8.2-4fb1a6c
    controller:
      image: fluidcloudnative/alluxioruntime-controller:v0.8.2-4fb1a6c
    runtime:
      image: fluidcloudnative/alluxio:release-2.8.1-SNAPSHOT-0433ade   #该部分需要根据上面构建的镜像修改
    fuse:
      image: fluidcloudnative/alluxio-fuse:release-2.8.1-SNAPSHOT-0433ade #该部分需要根据上面构建的镜像修改
```
用helm方式升级，如：
helm upgrade -n fluid-system --install fluid  ../fluid -f values.yaml

# s3对接，Ceph对象存储对接，CephFS对接
直接提供Demo放在自己的fluid仓库的example下：
## 1、S3对接
https://github.com/andyzheung/fluid/tree/master/samples/s3-exmaple
## 2、Ceph对象存储
https://github.com/andyzheung/fluid/tree/master/samples/ceph-object-storage-example

## 3、Cephfs对接
https://github.com/andyzheung/fluid/tree/master/samples/cephfs-cache
