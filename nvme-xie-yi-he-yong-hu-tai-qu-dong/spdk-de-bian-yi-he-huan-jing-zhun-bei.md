# SPDK的编译和环境准备

1. 下载 spdk 源码\
   git clone git@github.com:spdk/spdk.git
2. 更新依赖的库\
   cd spdk\
   bash scripts/pkgdep.sh\
   git submodule update --init
3. 编译源码\
   ./configure && make
4. 有可能发现少python module elftools，手动下载源码并安装\
   pyelftools-0.31.tar.gz\
   tar xvf pyelftools-0.31.tar.gz\
   cd pyelftools-0.31\
   python3 setup.py install
5. 如果编译的版本是spdk-21.10.x，有可能出现 dpdk struct imcomplete type，需要手动切换 dpdk 的版本到前几个
6. 编译完成后可以跑一下 spdk 的 hello world确保环境 ok\
   ./build/examples/hello\_world\
