## 下载源码

链接：https://github.com/redis/redis

git clone https://github.com/redis/redis

workspace目录中

第一条提交上新建分支study

检出study

## 编译

需要gcc和make

windows下载cygwin：https://www.cygwin.com/

当前最新版是3.2.0，

下载gcc和make

环境变量配置到C:\cygwin64\bin，

有一堆工具，其中有gcc.exe和make.exe

进入redis源码目录

运行make

成功编译生成了几个文件

运行redis服务端

.\redis-server.exe

成功运行

编译生成文件和源码混杂，乱

make clean

修改Makefile文件

可运行文件输出到bin目录

BIN = bin

PRGNAME = $(bin)/redis-server

BENCHPRGNAME = $(bin)/redis-benchmark

CLIPRGNAME = $(bin)/redis-cli

拆分清理操作

clean-o:

  @echo "清理.o文件"

  rm *.o

clean-bin:

  @echo "清理可运行文件"

  rm $(bin)/*

创建bin目录

再次make

在bin目录下生成exe文件

清理.o文件

make clean-o

每次make自动清理.o文件

all: redis-server redis-benchmark redis-cli clean-o

修改后Makefile文件全文

## 源码概览

文件

```
adlist.c
adlist.h
dict.c
dict.h
sds.c
sds.h

ae.c
ae.h
anet.c
anet.h
redis.c
redis-cli.c
zmalloc.c
zmalloc.h
```

## 程序流程

### 服务端启动流程

看redis.c

main方法

```c
int main(int argc, char **argv) {
    ...
    aeCreateFileEvent(server.el, server.fd, AE_READABLE,
        acceptHandler, NULL, NULL);
    ...
    aeMain(server.el);
    ...
}
```

```c
void aeMain(aeEventLoop *eventLoop)
{
    eventLoop->stop = 0;
    while (!eventLoop->stop)
        aeProcessEvents(eventLoop, AE_ALL_EVENTS);
}
```

```c
int aeProcessEvents(aeEventLoop *eventLoop, int flags)
{
    ...
    /* Check file events */
	...
    /* Note that we want call select() even if there are no
     * file events to process as long as we want to process time
     * events, in order to sleep until the next time event is ready
     * to fire. */
    ...
    retval = select(maxfd+1, &rfds, &wfds, &efds, tvp);
    ...
    /* Check time events */
    ...
}
```
再细分
```c
int aeProcessEvents(aeEventLoop *eventLoop, int flags)
{
    ...
    /* Check file events */
    ...
    fe = eventLoop->fileEventHead;
    ...
    fe->fileProc(eventLoop, fe->fd, fe->clientData, mask);
    ...
    /* Check time events */
    ...
    te = eventLoop->timeEventHead;
    ...
    retval = te->timeProc(eventLoop, id, te->clientData);
    ...
}
```

```c
static void acceptHandler(aeEventLoop *el, int fd, void *privdata, int mask) {
	...
    cfd = anetAccept(server.neterr, fd, cip, &cport);
	...
    createClient(cfd);
	...
}
```

### 客户端启动流程

```c
int main(int argc, char **argv) {
    ...
    return cliSendCommand(argc, argvcopy);
}
```

```c
static int cliSendCommand(int argc, char **argv) {
    ...
    fd = cliConnect();
    ...
    anetWrite(fd,cmd,sdslen(cmd));
    ...
    retval = cliReadInlineReply(fd,REDIS_CMD_INTREPLY);
    ...
}
```

### 网络IO流程

```c
static void acceptHandler(aeEventLoop *el, int fd, void *privdata, int mask) {
	...
    cfd = anetAccept(server.neterr, fd, cip, &cport);
	...
    createClient(cfd);
	...
}
```

```c
static redisClient *createClient(int fd) {
    ...
    aeCreateFileEvent(server.el, c->fd, AE_READABLE,
        readQueryFromClient, c, NULL);
    ...
}
```

```c
static void readQueryFromClient(aeEventLoop *el, int fd, void *privdata, int mask) {
    ...
    nread = read(fd, buf, REDIS_QUERYBUF_LEN);
	...
	processCommand(c);
    ...
}
```

```c
static int processCommand(redisClient *c) {
    ...
    cmd = lookupCommand(c->argv[0]->ptr);
    ...
    cmd->proc(c);
    ...
}
```

```c
static struct redisCommand *lookupCommand(char *name) {
    int j = 0;
    while(cmdTable[j].name != NULL) {
        if (!strcmp(name,cmdTable[j].name)) return &cmdTable[j];
        j++;
    }
    return NULL;
}
```

```c
static struct redisCommand cmdTable[] = {
	...
    {"ping",pingCommand,1,REDIS_CMD_INLINE},
	...
};
```

```c
static void pingCommand(redisClient *c) {
    addReply(c,shared.pong);
}
```

```c
static void addReply(redisClient *c, robj *obj) {
    ...
    aeCreateFileEvent(server.el, c->fd, AE_WRITABLE,
        sendReplyToClient, c, NULL);
    ...
}
```

```c
static void sendReplyToClient(aeEventLoop *el, int fd, void *privdata, int mask) {
    ...
	nwritten = write(fd, o->ptr+c->sentlen, objlen - c->sentlen);
    ...
}
```
### 




