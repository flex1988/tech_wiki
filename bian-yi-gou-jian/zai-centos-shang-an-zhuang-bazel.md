# 在centos上安装bazel

1. clone仓库git@github.com:sub-mod/bazel-builds.git\
   git clone git@github.com:sub-mod/bazel-builds.git
2. 如果没有docker，需要先安装docker\
   yum install -y docker\
   systemctl start docker
3. 下载docker镜像，准备bazel编译环境\
   docker build -t submod/bazel-build -f Dockerfile .
4. 进入docker，下载bazel源码，开始编译bazel\
   docker run -it -v $(pwd):/opt/app-root/src -u 0 submod/bazel-build:latest /bin/bash
5. 查找jdk路径\
   ![](../.gitbook/assets/image.png)
6. 设置JAVA\_HOME\
   export JAVA\_HOME=/data/docker/lib/overlay2/038e6d43d9aac97e029fd75be7563676c6eb1df19be9d0acdd5f20e03946f11c/diff/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.275.b01-0.el6\_10.x86\_64/

