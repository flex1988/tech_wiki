# Linux write每次最多能写多少字节的数据

背景

最近碰到一个线上的bug，具体情况是进程将24G左右的内存buffer写到磁盘上，但是每次write都写不完24G，然后重试导致磁盘写满，服务不可用。

在修这个bug的时候，对于到底write最大能写多少字节的数据产生了浓厚的兴趣，写了一个测试程序基本上每次最多写到2G多一点。

在研究了一段内核代码后一切霍然开朗。

#### write限制

write函数在以下三种情况下可能写入的字节数小于指定的字节数：

1. 在底层的物理介质上没有足够的空间
2. RLIMIT\_FSIZE的限制
3. 写入被信号打断

从内核代码来看，count在大于MAX\_RW\_COUNT的情况下，会赋值为MAX\_RW\_COUNT

而MAX\_RW\_COUNT是一个宏，展开为：INT\_MAX & PAGE\_MASK

INT\_MAX也是一个宏，展开为((int)(\~0U>>1))，也就是无符号数0取反后右移一位转换成int类型，也就是2^31.

PAGE\_MASK也是一个宏，展开为(\~(PAGE\_SIZE-1))，而PAGE\_SIZE展开为(\_AC(1,UL) << PAGE\_SHIFT)，PAGE\_SHIFT的值为12，也就是每页的大小是2^12，也就是说1左移12位，PAGE\_SIZE的值为2^12，然后PAGE\_SIZE-1取反

最后MAX\_RW\_COUNT的值为INT\_MAX & PAGE\_MASK，也就是说MAX\_RW\_COUNT的值是int的最大值最后12位屏蔽掉，保持4K地址对齐

所以理论上讲，每次write可写的buff大小是2^31-2^12=2147479552，这也是与实际的测试结果相一致

#### 测试代码

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <fcntl.h>
#include <sys/resource.h>
#include <signal.h>
#include <errno.h>
#include <string.h>
void sigsegvHandler(int sig, siginfo_t *info, void *secret) {
	printf("%d %s",sig,secret);
}
void sigtermHandler(int sig) {
	printf("%d\n",sig);
}

void setupSignalHandlers(void) {
    struct sigaction act;

    sigemptyset(&act.sa_mask);
    act.sa_flags = 0;
    act.sa_handler = sigtermHandler;
    sigaction(SIGTERM, &act, NULL);

    sigemptyset(&act.sa_mask);
    act.sa_flags = SA_NODEFER | SA_RESETHAND | SA_SIGINFO;
    act.sa_sigaction = sigsegvHandler;
    sigaction(SIGSEGV, &act, NULL);
    sigaction(SIGBUS, &act, NULL);
    sigaction(SIGFPE, &act, NULL);
    sigaction(SIGILL, &act, NULL);
    return;
}

int main(int argc,char** argv){
	signal(SIGHUP, SIG_IGN);
    	signal(SIGPIPE, SIG_IGN);
    	setupSignalHandlers();
	//extern int errno;
	struct rlimit limit;
	limit.rlim_cur = RLIM_INFINITY;
	limit.rlim_max = RLIM_INFINITY;
	if(setrlimit(RLIMIT_FSIZE,&limit)){
		printf("set limit failed\n");
	}
	long long buff_size = atoll(argv[1]);
	char *buff = malloc(buff_size*sizeof(char));
	int fd = open("io.dat", O_CREAT|O_WRONLY|O_TRUNC, 0644);
	int ret = 0;
	while(buff_size>0){
	ret = write(fd,buff,buff_size);
	printf("written %d bytes\n",ret);
	if(ret<0){
		printf("write error: %s\n",strerror(errno));
		return 0;
	}
	buff_size-=ret;
	if(buff_size==0) break;
	}
 	printf("write %lld bytes\n",buff_size);
	return 0;
}
```
