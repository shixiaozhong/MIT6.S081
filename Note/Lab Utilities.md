[sleep(<font color="green">easy</font>)](#sleep(<font color="green">easy</font>))

[pingpong(<font color="green">easy</font>)](#pingpong(<font color="green">easy</font>))

[primes (<font color="blue">moderate</font>>)/(<font color="red">hard</font>)](#primes (<font color="blue">moderate</font>>)/(<font color="red">hard</font>))

[find (<font color="blue">moderate</font>)](#find (<font color="blue">moderate</font>))

#### sleep(<font color="green">easy</font>)

**要求**

> Implement the UNIX program `sleep` for xv6; your `sleep` should pause for a user-specified number of ticks. A tick is a notion of time defined by the xv6 kernel, namely the time between two interrupts from the timer chip. Your solution should be in the file `user/sleep.c`.          
>
> 实现给xv6提供的UNIX程序sleep，你的sleep需要暂停用户指定的刻度数，刻度是由 xv6内核定义的时间概念，是在定时芯片的两个中断之间的时间，你的答案需要在`user/sleep.c`文件中。

**一些提示**

* 在编写代码之前，你需要去看 [xv6 book](https://pdos.csail.mit.edu/6.828/2022/xv6/book-riscv-rev3.pdf)的第一章。
* 去看user目录下的其他代码(例如 `user/echo.c`, `user/grep.c`, and `user/rm.c`)，去看如何获得传入程序的命令行参数。
* 如果用户忘记传参数了，`sleep`需要给予一个错误信息
* 命令行的参数是通过字符串来传递的，你可以使用`atoi`函数(在`user/ulib.c`)来将其转换为整型。
* 使用系统调用sleep。
* 去`kernel/sysproc.c`看xv6的内核实现sleep系统调用的代码，`user/user.h`提供了从用户程序调用的sleep的C义,`user/usys/.S`提供了从用户代码跳转到内核的进行睡眠的汇编代码。
* 当main函数完成时需要调用`exit(0)`。
* 将你的sleep程序添加到Makefile下的UPORGS，一旦你这么做了， `make qemu`将会编译你的程序，你就可以在xv6的shell中跑。
* 去看Kernighan and Ritchie's book的c语言程序设计，去学习c语言。

在xv6的shell下跑程序：

![image-20221101230733534](http://shixiaozhong.oss-cn-hangzhou.aliyuncs.com/img/image-20221101230733534.png)

通过`make grade`来测试代码正确性，但是`make grade`会跑所有的测试，如果只需要跑一个任务的测试可以使用`$ ./grade-lab-util sleep`来测试sleep的代码正确性。或者使用`$ make GRADEFLAGS=sleep grade`来进行测试。

```c
// sleep.c

#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main(int argc, char *argv[])
{
    // 判断参数是否小于两个，如果小于两个说明没有带参数
    if (argc < 2)
    {
        fprintf(1, "Usage: sleep seconds...\n");
        exit(1);
    }
    // 判断输入的数字不能为负数
    if (*argv[1] != '-')
    {
        int n = atoi(argv[1]); // 字符串转化为数字
        sleep(n);              // 调用系统调用函数sleep
    }
    else
    {
        fprintf(1, "seconds must be positive\n"); // 输入的数字不是正数就饿给提示
        exit(2);
    }
    exit(0);
}
```

![image-20221101233538084](http://shixiaozhong.oss-cn-hangzhou.aliyuncs.com/img/image-20221101233538084.png)

---

#### pingpong(<font color="green">easy</font>)

**要求**

> Write a program that uses UNIX system calls to ''ping-pong'' a byte between two processes over a pair of pipes, one for each direction. The parent should send a byte to the child; the child should print "<pid>: received ping", where <pid> is its process ID, write the byte on the pipe to the parent, and exit; the parent should read the byte from the child, print "<pid>: received pong", and exit. Your solution should be in the file `user/pingpong.c`.
>
> 编写一个程序，使用 UNIX 系统调用通过一对pipe在两个进程之间“ ping-pong”一个字节，每个pipe对应一个方向。父亲应该发送一个字节给孩子，孩子需要打印“<pid>: received ping”，pid是它的进程id，往pipe中给父亲写一个字节，然后exit。父亲需要从pipe中读出来自孩子的那个字节，打印“<pid>: received ping”。你的答案需要在`user/pingpong.c`文件中。

**一些提示**	

* 使用`pipe`创建一个管道
* 使用`fork`创建一个子进程
* 使用`read`从一个pipe里读，使用write从一个pipe里写。
* 使用`getpid`查找一个进程的id号
* 将程序加入到Makefile中的UPROGS
* Xv6上的用户程序只能使用有限的库函数集，你可以在`user/user.h`中看到它们的列表，源码在`user/ulib.c`,`user/printf.c`和`user/umalloc.c`中。

在xv6的shell下跑，它应该产生一下输出。

![image-20221102085134258](http://shixiaozhong.oss-cn-hangzhou.aliyuncs.com/img/image-20221102085134258.png)

```c
//pingpong.c

#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main(int argc, char *argv[])
{
    int p1[2], p2[2];
    pipe(p1);         // 创建pipe1
    pipe(p2);         // 创建pipe2
    int pid = fork(); // 创建子进程
    if (pid == 0)
    {
        // child process
        // 子进程从pipe1写， 从pipe2读
        close(p1[0]); // 关闭pipe1读端
        close(p2[1]); // 关闭pipe2写端

        char buf[4];
        read(p2[0], buf, sizeof(buf)); // 从pipe2中读数据
        fprintf(1, "%d: received ping\n", getpid());

        write(p1[1], buf, sizeof(buf)); // 往pipe1中写数据
        exit(1);
    }
    else if (pid > 0)
    {
        // parent process
        char s[4] = "hhh";
        write(p2[1], s, 4); // 往pipe2中写数据, 在等待之前写数据，如果在等待之后写，就会导致子进程一直等待读数据
                            // 如果没有数据可以读，读端进程会被挂起，知道有数据可以读
        int status;
        pid = wait(&status); // wait child，阻塞式的等待
        // 父进程从pipe1读，从pipe2写
        close(p1[1]); // 关闭pipe1写端
        close(p2[0]); // 关闭pipe2读端

        char buf[4];
        read(p1[0], buf, sizeof(buf)); // 从pipe1中读数据
        fprintf(1, "%d: received pong\n", getpid());
    }
    else
    {
        fprintf(1, "fork failed\n");
        exit(1);
    }
    exit(0);
}
```

![image-20221102152144556](http://shixiaozhong.oss-cn-hangzhou.aliyuncs.com/img/image-20221102152144556.png)

---

#### primes (<font color="blue">moderate</font>>)/(<font color="red">hard</font>)

**要求**

> Write a concurrent version of prime sieve using pipes. This idea is due to Doug McIlroy, inventor of Unix pipes. The picture halfway down [this page](http://swtch.com/~rsc/thread/) and the surrounding text explain how to do it. Your solution should be in the file `user/primes.c`.
>
> 用管道来写一个并发版本的质数筛，这个想法来自于 Unix 管道的发明者 Doug McIlroy。在[this page](http://swtch.com/~rsc/thread/)中间的图片和周围的文字解释了如何做到这一点。你的答案应该放在`user/primes.c`文件中。
>
> Your goal is to use `pipe` and `fork` to set up the pipeline. The first process feeds the numbers 2 through 35 into the pipeline. For each prime number, you will arrange to create one process that reads from its left neighbor over a pipe and writes to its right neighbor over another pipe. Since xv6 has limited number of file descriptors and processes, the first process can stop at 35.
>
> 你的目标是使用pipe和fork来创建pipeline。第一个进程将数字2至35的数字放到管道中。对于每一个质数，你将安排创建一个进程通过pipe从它的左边邻居读取，通过另一个pipe写到它的右邻居。由于Xv6的文件描述符和进程数量有限，第一个进程可以在35时停止。

**一些提示**

* 小心关闭进程不需要的文件描述符，因为，您的程序可能将在第一个进程到达35之前用完Xv6的资源运行。

* 一旦第一个进程达到35，它应该等待直到整个pipeline终止。包括所有的子孙后代进程。因此，主素数进程只应在所有输出都打印出来之后，以及在所有其他素数进程都退出之后才退出。

* 当一个管道的写端被关闭时，read函数返回0。

* 将32位(4字节) int 直接写入管道是最简单的，而不是使用格式化的ASCII I/O。

* 您应该只在需要时在管道中创建进程。
* 将程序加入到Makefile中的UPROGS。

如果你的代码实现了基于管道的筛选器，并且满足以下输出，那么它就是正确的。**开始做题的时候需要看上面提到的那篇文章，思路就在那篇文章中。**

<img src="http://shixiaozhong.oss-cn-hangzhou.aliyuncs.com/img/image-20221102162822444.png" alt="image-20221102162822444"  />

```c
// primes.c

#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

void prime_sieve(int *p_left)
{
    // 读出第一个数据，判断是否标志位
    int q;
    read(p_left[0], &q, sizeof(q));
    if (q == -1) // 如果第一个数据表示-1，则数据全部筛完，进程结束。
        exit(0);
    fprintf(1, "prime %d\n", q); // 否则将筛出的质数打印到屏幕

    int p_right[2];
    pipe(p_right);    // 创建与子进程通信的右边管道
    int pid = fork(); // 创建一个子进程
    if (pid == 0)
    {
        // child process
        close(p_right[1]); // 关闭掉与父进程通信的管道的写端
        prime_sieve(p_right);
    }
    else
    {
        // parent process
        close(p_right[0]); // 关闭与子进程通信管道的读端
        int p;
        while (read(p_left[0], &p, sizeof(p))) // 从管道中将父进程写入的数据读出来,最后一个标志位舍去
        {
            // 读到标志位后break
            if (p == -1)
            {
                write(p_right[1], &p, sizeof(p)); // 将标志位写入
                break;
            }
            // 判断是否可以被整除，可以代表一定不是素数
            if (p % q != 0)
            {
                write(p_right[1], &p, sizeof(p)); // 写入到与子进程通信的管道中，让子进程继续筛选
            }
        }
        wait(0);          // 等待子进程退出
    }
}

int main()
{
    int p[2];
    pipe(p); // 创建管道
    int pid = fork();
    if (pid == 0)
    {
        // child process
        close(p[1]);    // 关闭写端口
        prime_sieve(p); // 子进程利用与左侧进程的管道读端口
    }
    else
    {
        // parent process
        close(p[0]); // 关闭读端口
        for (int i = 2; i < 36; i++)
            write(p[1], &i, sizeof(i)); // 将2-35全部读入到管道中
        int flag = -1;
        write(p[1], &flag, sizeof(flag)); // 设置一个标志位，标志数据最后一位
        wait(0);                          // 等待子进程退出
    }
    exit(0);
}
```

![image-20221102181839630](http://shixiaozhong.oss-cn-hangzhou.aliyuncs.com/img/image-20221102181839630.png)

---

#### find (<font color="blue">moderate</font>)

**要求**

> Write a simple version of the UNIX find program: find all the files in a directory tree with a specific name. Your solution should be in the file `user/find.c`.
>
> 写一个简单版本的UNIX find程序：查找目录树中具有特定名称的所有文件。你的答案应该放在`user/find.c`文件中。

**一些提示**

* 去`user/ls.c`中查看如何读目录。
* 使用递归允许 find 下降到子目录。
* 不要递归到“ .”和“ . .”。
* 文件系统的更改在 qemu 运行期间保持不变，为了得到一个干净的文件系统，先执行`make clean`，然后执行`make qemu`。
* 你需要使用C的字符串，看一下K&RC语言书籍。
* 注意`==`并不是像python一样判断字符串，使用`strcmp`代替。
* 将程序加入到Makefile中的UPROGS。
