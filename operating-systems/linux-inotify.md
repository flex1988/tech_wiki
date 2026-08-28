# Linux inotify

### 1. inotify功能

Linux inotify是一个内核用于通知程序文件系统变化的机制。

开源社区提出用户态需要内核提供一些机制，以便用户态能够及时得知内核或者底层硬件设备发生的事件，从而能够更好的管理设备，给用户提供更好的服务，比如hotplug、udev和inotify都是这种需求催生的。

inotify是文件系统变化通知机制，在监听到文件系统变化后，会向相应的应用程序发送事件。典型的应用场景是文件管理器，理想情况下是用户修改了文件内容后立刻显示出文件最新的内容。

### 2. 用户接口

在用户态，inotify通过3个系统调用和在返回的文件描述符上的文件操作来使用，使用inotify的第一步是创建inotify实例：

int fd = inotify\_init();

每一个inotify实例对应一个独立的排序的队列。文件系统的变化事件被称作watches的一个对象管理，每一个watch是一个二院组{目标， 时间掩码}，目标可以是文件或目录，事件掩码表示应用希望关注的inotify事件，每一个bit对应一个inotify事件，watch对象通过watch描述符引用，watchs通过文件或者目录的路径名来添加。目录watches将返回在该目录下的所有文件上面发生的事件。

下面函数用于添加一个watch:

int wd = inotify\_add\_watch(fd, path, mask);

fd是inofity\_init()返回的文件描述符，path是被监视的目标的路径名（即文件名或目录名），mask是事件掩码，在头文件linux/inotify.h中定义了每一位代表的事件，改变事件掩码就可以改变希望被通知的inotify事件。

下面函数被用于删除一个watch:

int ret = inotify\_rm\_watch(fd, wd);

fd是inotify\_init返回的文件描述符，wd是inotify\_add\_watch返回的Watch描述符。

文件事件用一个inotify\_event结构表示，它通过由inotify\_init返回的文件描述符使用通常文件读取函数read来获得

```
struct inotify_event {
    __s32    wd;        /* watch descriptor */
    __u32    mask;      /* watch mask */
    __u32    cookie;    /* cookie to synchronize two events */
    __u32    len;       /* length of name */
    char     name[0];   /* stub for possible name */
};
```

结构中的wd是监视目标的watch fd，mask事件掩码，len是name字符串的长度，name是监视目标的路径名，该结构的name字段为一个桩，它只是为了用户方面引用文件名，文件名是变长的，它实际紧跟在该结构的后面，文件名将被0填充以使下一个事件结构能够4字节对齐。

通过read调用一次可以获得多个事件

size\_t len = read(fd, buf, BUF\_LEN);

buf是一个inotify\_event结构的数组指针，BUF\_LEN指定要读取的总长度，buf大小至少要不小于BUF\_LEN，该调用返回的事件数取决于BUF\_LEN以及事件中文件名的长度。

### 3. 事件列表

IN\_ATTRIB 文件属性被修改

IN\_CLOSE\_WRITE 可写文件被关闭

IN\_CLOSE\_NOWRITE 不可写文件被关闭

IN\_CREATE 文件、文件夹被创建

IN\_DELETE 文件、文件夹被删除

IN\_DELETE\_SELF 被监控的对象本身被删除

IN\_MODIFY 文件被修改

IN\_MOVE\_SELF 被监控的对象被move

IN\_MOVED\_FROM 文件被移出被监控目录\
IN\_MOVED\_TO 文件被移入被监控目录

IN\_OPEN 文件被打开



