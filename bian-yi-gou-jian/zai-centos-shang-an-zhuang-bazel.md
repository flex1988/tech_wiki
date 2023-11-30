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
5. 安装jdk\
   yum install java\
   yum install java-devel
6. 查找jdk路径\
   ![](../.gitbook/assets/image.png)
7. 设置JAVA\_HOME\
   export JAVA\_HOME=/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.392.b08-2.tl2.x86\_64/

