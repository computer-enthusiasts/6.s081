



​	在Makefile中把你写的程序(如)添加到UPROGS中；一旦你这样做了，make qemu将编译你的程序，你就可以从xv6 shell中运行它。

```c
//primes.c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

void primes(int *fd){
    int p, d;
    int status1;
    // 关闭写
    close(fd[1]);

    if(read(fd[0], (void *)&p, sizeof(p)) != sizeof(p)){
        fprintf(2, "Read fail!\n");
        exit(1);
    }
    printf("prime %d\n", p);

    if (read(fd[0], (void *)&d, sizeof(d))){//从fd管道取数
        int fd1[2];
        pipe(fd1);

        if (fork() == 0){//child
            primes(fd1);
        }else{
            // 关闭读
            close(fd1[0]);
            do{
                if (d % p != 0){
                    write(fd1[1], &d, sizeof(d));//放入fd1管道右边
                }
            }while(read(fd[0], &d, sizeof(d)));//管道中还有数
           
            close(fd[0]); // 关闭读
            close(fd1[1]); // 关闭写
            
            wait(&status1);
        }
    }
    exit(0);
}

int main(int argc, char *argv[]){
    int fd[2];
    int start = 2;
    int end = 35;
    pipe(fd);
    int status;
    if (fork() == 0){
        primes(fd);
    }else{
        close(fd[0]);
        for (int i = start; i <= end; i++){
          //write函数把buf中nbyte写入文件描述符handle所指的文档，成功时返回写的字节数，错误时返回-1.
            if (write(fd[1], &i, sizeof(i)) != 4){ // 生成 [2, 35]，输入管道链最左端,int是4字节
                fprintf(2, "Write fail!\n");
                exit(1);
            }
        }
        close(fd[1]);
        wait(&status);//或者直接wait(0),等待所有子进程结束
    }
    
    exit(0);
}
```



​	使用多进程和管道，每一个进程作为一个 stage，筛掉某个素数的所有倍数（筛法）。很巧妙的形式实现了多线程的筛法求素数。(参考本篇:[Bell Labs and CSP Threads](https://swtch.com/~rsc/thread/)).

​	主进程：生成 n ∈ [2,35] -> 子进程1：筛掉所有 2 的倍数 -> 子进程2：筛掉所有 3 的倍数 -> 子进程3：筛掉所有 5 的倍数 -> .....(每创建一个子进程,就会创建一个新管道,该子进程的输出端连着管道的读描述符)

​	每一个 stage 以当前数集中最小的数字作为素数输出（每个 stage 中数集中最小的数一定是一个素数，因为它没有被任何比它小的数筛掉），并筛掉输入中该素数的所有倍数（必然不是素数），然后将剩下的数传递给下一 stage。最后会形成一条子进程链，而由于每一个进程都调用了 `wait(0);` 等待其子进程，所以会在最末端也就是最后一个 stage 完成的时候，沿着链条向上依次退出各个进程。

​	这一道的主要坑就是，stage 之间的管道 pleft 和 pright，要注意关闭不需要用到的文件描述符，否则跑到 n = 13 的时候就会爆掉，出现读到全是 0 的情况。

​	这里的理由是，fork 会将父进程的所有文件描述符都复制到子进程里，而 xv6 每个进程能打开的文件描述符总数只有 16 个 （见 `defs.h` 中的 `NOFILE` 和 `proc.h` 中的 `struct file *ofile[NOFILE]; // Open files`）。

​	由于一个管道会同时打开一个输入文件和一个输出文件，所以**一个管道就占用了 2 个文件描述符**，并且复制的**子进程还会复制父进程的描述符**，于是跑到第六七层后，就会由于最末端的子进程出现 16 个文件描述符都被占满的情况，导致新管道创建失败。

解决方法有两部分：

- 关闭管道的两个方向中不需要用到的方向的文件描述符（在具体进程中将管道变成只读/只写）

  > 原理：每个进程从左侧的读入管道中**只需要读数据**，并且**只需要写数据**到右侧的输出管道，所以可以把左侧管道的写描述符，以及右侧管道的读描述符关闭，而不会影响程序运行 这里注意文件描述符是进程独立的，在某个进程内关闭文件描述符，不会影响到其他进程

- 子进程创建后，关闭父进程与祖父进程之间的文件描述符（因为子进程并不需要用到之前 stage 的管道）











find（适度）

​	编写一个简单版本的UNIX find程序：在一个目录树中找到所有具有特定名称的文件。你的解决方案应该在文件user/find.c中。

some hints:

    看看user/ls.c以了解如何读取目录。
    使用递归来允许查找进入子目录。
    不要递归到". "和".."。
    对文件系统的改变会在qemu运行期间持续存在；要获得一个干净的文件系统，先运行make clean，然后再运行qemu。
    你需要使用 C 字符串。请看K&R（C语言书），例如第5.5节。
    注意，==不能像Python中那样比较字符串。用strcmp()代替。
    将该程序添加到Makefile中的UPROGS。

如果产生以下输出，你的方案就是正确的（当文件系统包含文件b和a/b时）。

```c
$ make qemu
...
init: starting sh
$ echo > b  //如果刚开始什么都没有,则会建立一个b的txt(>是重定向符,但输入是空白)
$ mkdir a	//创立一个文件夹,叫a
$ echo > a/b //文件夹a里出现了一个b的txt文本
$ find . b  //查找目录和文件包含.或b的
./b
./a/b
$ 
```



solution:

需要使用ls.c中的代码，整体思路如下：

- 判断当且路径是否为文件，如果是文件则直接判断文件名是否和目标文件名相同。
- 否则当且路径为目录，遍历目录下文件，递归处理。



```

```

