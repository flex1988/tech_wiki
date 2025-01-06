# LIBRARY\_PATH和LD\_LIBRARY\_PATH

LIBRARY\_PATH和LD\_LIBRARY\_PATH是Linux下的两个环境变量，二者的含义和作用分别如下：

LIBRARY\_PATH环境变量用于在gcc在链接，编译，生成obj期间查找动态链接库时指定查找共享库的路径，除了标准的路径/usr/lib,/usr/local/lib，ld也在LIBRARY\_PATH指定的目录里查找动态库。

LD\_LIBRARY\_PATH环境变量用于&#x5728;_**程序加载运行期间**_&#x67E5;找动态链接库时指定除了系统默认路径之外的其他路径，注意，LD\_LIBRARY\_PATH中指定的路径会在系统默认路径之前进行查找。

