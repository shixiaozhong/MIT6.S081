[sleep(<font color="green">easy</font>)](#sleep(<font color="green">easy</font>))

[pingpong(<font color="green">easy</font>)](#pingpong(<font color="green">easy</font>))

[primes (<font color="blue">moderate</font>>)/(<font color="red">hard</font>)](#primes (<font color="blue">moderate</font>>)/(<font color="red">hard</font>))

[find (<font color="blue">moderate</font>)](#find (<font color="blue">moderate</font>))

[xargs (<font color="blue">moderate</font>)](#xargs (<font color="blue">moderate</font>))

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

**思路**

这个就是实现一个sleep的命令，直接调用系统自带的sleep函数就可以。

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
    int n = atoi(argv[1]);	// 将字符串转换为数字
    sleep(n);	// 执行内置的系统调用进行sleep
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

**思路**

实现pingpong命令，本质就是通过管道来实现进程之间的通信，由于管道是半双工，不能同时发，并且是单向通信，所以需要用一对，两个管道来完成通信，一个负责读，一个负责写。

```c
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
        fprintf(1, "%d: received %s\n", getpid(), buf);
        char s[5] = "pong";
        write(p1[1], s, sizeof(buf)); // 往pipe1中写数据
        exit(1);
    }
    else if (pid > 0)
    {
        // parent process
        char s[5] = "ping";
        write(p2[1], s, 4); // 往pipe2中写数据, 在等待之前写数据，如果在等待之后写，就会导致子进程一直等待读数据
                            // 如果没有数据可以读，读端进程会被挂起，知道有数据可以读
        int status;
        pid = wait(&status); // wait child，阻塞式的等待

        // 父进程从pipe1读，从pipe2写
        close(p1[1]); // 关闭pipe1写端
        close(p2[0]); // 关闭pipe2读端

        char buf[4];
        read(p1[0], buf, sizeof(buf)); // 从pipe1中读数据
        fprintf(1, "%d: received %s\n", getpid(), buf);
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

**思路**

可以先去看看那篇文章，文章中的图就提供了思路来实现，这个素数筛的实现应该是最难，应该也是最有意思的一个实现。

![image-20221113212115496](http://shixiaozhong.oss-cn-hangzhou.aliyuncs.com/img/image-20221113212115496.png)

文章中的思路已经很明显了，找到第一个素数2，将其打印，然后将所有是2倍数的数全部筛掉，然后继续下一个，第一个素数为3，同同样的打印，然后将所有的3的倍数的数全部筛掉，一直往后操作，直到没有数字的为止。

思路有了，怎么实现呢？提示中也有，就是通过子进程来完成筛的动作，并且进程之间还要有通信，因为需要传递数据。所以还需要用到管道，但是可以注意到只需要从父进程读取数据，不需要写入，所以只需要一个管道就可以完成工作。

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
            // 读到标志位后写入，然后break
            if (p == -1)
            {
                write(p_right[1], &p, sizeof(p)); // 将标志位写入
                break;
            }
            // 判断是否可以被整除，可以代表一定不是素数
            if (p % q != 0)
            {
                write(p_right[1], &p, sizeof(p)); // 不可以被整除的写入到与子进程通信的管道中，让子进程继续筛选
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
        write(p[1], &flag, sizeof(flag)); // 设置一个标志位，标志有效数据结束
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

**思路**

find的查找思路还是比较简单的，就是查看文件，如果待查找的是文件，直接判断，如果是目录就去目录中的每一个文件中去查找，所以可以想到用递归来完成查找。但是细节比较多，比如如何读取目录下的每一个文件名，查看文件状态，可以按照提示去查看`user/ls.c`是如何进行的，还有一些字符串的连接和判断，对指针的应用还是比较麻烦的。

```c
// ls.c

#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"

char *
fmtname(char *path)
{
  static char buf[DIRSIZ + 1];
  char *p;

  // Find first character after last slash.
  for (p = path + strlen(path); p >= path && *p != '/'; p--)	// 从后往前找第一个分隔符
    ;
  p++;	// p++以后p指向的就是路径中最后一个文件的名称的首地址

  // Return blank-padded name.
  if (strlen(p) >= DIRSIZ)	// 如果最后一个文件的长度超过目录长度的最大值，直接返回p
    return p;
  memmove(buf, p, strlen(p));	// 否则将最后一个文件名拷贝到buf中，此时buf中只存储了一个文件名
  memset(buf + strlen(p), ' ', DIRSIZ - strlen(p));	//将文件的长度补为DIRSIZ，后面补空格，主要目的是保证打印对齐
  return buf;
}

void ls(char *path)
{
  char buf[512], *p;
  int fd;				// 当前文件描述符
  struct dirent de;		// 存储文件索引和名称
  struct stat st;		// 保存文件描述信息

  if ((fd = open(path, 0)) < 0)		// 打开文件
  {
    fprintf(2, "ls: cannot open %s\n", path);
    return;
  }

  if (fstat(fd, &st) < 0)	// 获取文件状态
  {
    fprintf(2, "ls: cannot stat %s\n", path);
    close(fd);
    return;
  }

  switch (st.type)	// 判断文件状态
  {
  case T_DEVICE:
  case T_FILE:		// 是文件
    printf("%s %d %d %l\n", fmtname(path), st.type, st.ino, st.size);
    break;

  case T_DIR:		// 是目录
    if (strlen(path) + 1 + DIRSIZ + 1 > sizeof buf)	// 路径长度超过buf
    {
      printf("ls: path too long\n");
      break;
    }
    strcpy(buf, path);
    p = buf + strlen(buf);
    *p++ = '/';				// p存储文件路径，且最后一位置为分隔符/
    while (read(fd, &de, sizeof(de)) == sizeof(de))	// 读取每一个文件
    {
      if (de.inum == 0)
        continue;
      memmove(p, de.name, DIRSIZ);		// memmove函数将目录名和路径连接起来
      p[DIRSIZ] = 0;					// 最后一位置0
      if (stat(buf, &st) < 0)			// 获取遍历文件的状态
      {
        printf("ls: cannot stat %s\n", buf);
        continue;
      }
      printf("%s %d %d %d\n", fmtname(buf), st.type, st.ino, st.size);	// 打印文件状态
    }
    break;
  }
  close(fd);
}

int main(int argc, char *argv[])
{
  int i;

  if (argc < 2)
  {
    ls(".");	// 如果只输入一个ls，就去ls当前目录
    exit(0);
  }
  for (i = 1; i < argc; i++)
    ls(argv[i]);	// 否则ls每一个参数
  exit(0);
}

```

```c
// find.c

#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"
#include "kernel/fs.h"

void find(char *path, char *file)
{
    // fprintf(1, "路径：%s\n", path);
    struct stat st; // 存储文件描述信息
    int fd;
    if ((fd = open(path, 0)) < 0) // 打开当前文件
    {
        fprintf(2, "find: cannot open %s\n", path);
        return;
    }
    if (fstat(fd, &st) < 0) //获取当前文件状态
    {
        fprintf(2, "ls: cannot stat %s\n", path);
        close(fd);
        return;
    }
    if (st.type == T_DIR) // 是目录，继续向下搜索
    {
        char buf[512]; // 储存文件路径，假设最长不超过512字节
        strcpy(buf, path);
        char *p = buf + strlen(buf);
        *p++ = '/';                                                 // 将路径中最后的0覆盖，且置为路径分隔符
        struct dirent dir_dt;                                       // 存储目录下的文件名和编号
        while (read(fd, &dir_dt, sizeof(dir_dt)) == sizeof(dir_dt)) // 遍历每一个文件
        {
            if (dir_dt.inum == 0)
                continue;
            if (strcmp(dir_dt.name, ".") == 0 || strcmp(dir_dt.name, "..") == 0) // 不要递归.和..
                continue;
            strcpy(p, dir_dt.name); // 连接文件名，构成新路径
            find(buf, file);        // 继续寻找
        }
    }
    else if (st.type == T_FILE) // 是文件，直接判断是不是需要查找的文件名
    {
        // 从路径中取最后一个文件的名称
        char *p;
        for (p = path + strlen(path); p >= path && *p != '/'; p--)
            ;
        p++;                      // 使p指向文件名的首地址
        if (strcmp(p, file) == 0) // 判断相等，是就将路径打印
            fprintf(1, "%s\n", path);
    }
    close(fd); // 关闭文件描述符
}

int main(int argc, char *argv[])
{
    if (argc < 3)
    {
        fprintf(1, "Usge find directory-name file-name\n");
        exit(1);
    }
    find(argv[1], argv[2]); // 参数为目录名和文件名
    exit(0);
}
```

![image-20221104124939476](http://shixiaozhong.oss-cn-hangzhou.aliyuncs.com/img/image-20221104124939476.png)

---

#### xargs (<font color="blue">moderate</font>)

**要求**

> Write a simple version of the UNIX xargs program: its arguments describe a command to run, it reads lines from the standard input, and it runs the command for each line, appending the line to the command's arguments. Your solution should be in the file `user/xargs.c`.
>
> 写一个简单版本的UNIX xargs程序：它的参数描述了一个需要运行的命令，它从标准输入中读取行，并且它为每一行运行命令，将该行追加到命令的参数。你的答案应该放在`user/xargs.c`文件中。

下面的例子说明了 xarg 的行为:

![image-20221104222839937](http://shixiaozhong.oss-cn-hangzhou.aliyuncs.com/img/image-20221104222839937.png)

注意，这里的命令是“ echo bye”，附加参数是“ hello too”，使用“ echo bye hello too”命令，输出“ bye hello too”。

请注意，UNIX 上的 xargs 进行了一次优化，它将在一个时间内向命令提供多于参数的信息。我们不期望你做这个优化，让 UNIX 上的 xargs 按照我们希望的方式运行。请将-n 选项设置为1运行它。比如说：

![image-20221104223142242](http://shixiaozhong.oss-cn-hangzhou.aliyuncs.com/img/image-20221104223142242.png)

**一些提示**

* 使用 fork 和 exec 在每行输入中调用命令，在父进程中使用 wait 等待子进程完成命令。
* 若要读取单个输入行，请一次读取一个字符，直到出现换行符('\n')。
* 在`kernel/param.h`中声明了MAXARG，如果需要声明 argv 数组，它可能很有用。
* 将程序加入到Makefile中的UPROGS。
* 文件系统的更改在 qemu 运行期间保持不变，为了得到一个干净的文件系统，先执行`make clean`，然后执行`make qemu`

Xargs、 find 和 grep 组合得很好:

![image-20221104223626186](http://shixiaozhong.oss-cn-hangzhou.aliyuncs.com/img/image-20221104223626186.png)

将对下面目录中名为 b 的每个文件运行“ grep hello”。

**测试**

为了测试你的xargs 答案,运行 shell 脚本 xargstest.sh。如果您的解决方案产生以下输出，则该解决方案是正确的:

![image-20221104223747831](http://shixiaozhong.oss-cn-hangzhou.aliyuncs.com/img/image-20221104223747831.png)

您可能必须返回并修复 find 程序中的 bug。输出中有许多`$`，因为 xv6 shell 没有意识到它是从一个文件而不是从控制台处理命令,为文件中的每个命令打印 $。(这个bug暂时不考虑)

**思路**

xargs的作用就是从标准输入流中读取数据，并将其转化为后面指令的命令行参数。只需要从stdin中读出数据，直接作为后面指令的参数就可以了，然后创建一个子进程套exec函数。有一个细节需要注意，就是因为管道两边的两条指令都是同时执行的，所以需要关注到先后的问题，存在前面的数据还没有得到，xargs就直接开始运行的可能。下面写了两种写法，第一种是读一次，使用sleep显示等待，第二种是一次读一个字符，一直读。我也试过一直读一个buf数组的做法，但是不知道为啥不成功。

```c
#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main(int argc, char *argv[])
{
    sleep(2);	// 等待前面的命令先执行完，可以使用sleep
    
    char buf[1024]; // 存储从标准输入流读出的字符串
    char *xargv[16];	// 作为exec的第二个参数
    int xargc = 0;
    // 将argv的所有参数存储到xargv中，作为后续调用exec使用
    for (int i = 1; i < argc; i++)
        xargv[xargc++] = argv[i];
    
    char *p = buf; // p指向buf首地址
    read(0, buf, sizeof(buf));	// 从标准输入中读取数据到buf数组，只读一次
    for (int i = 0; i < sizeof(buf); i++)
    {
        if (buf[i] == '\n')	// 读到换行符代表一次读入已经结束
        {
            int pid = fork();
            if (pid == 0)
            {
                // child process
                // 将buf的参数加到xargv中
                buf[i] = '\0';         // 将'\n'转换为'\0'，便于连接
                xargv[xargc++] = p;    // 进行连接
                xargv[xargc++] = '\0'; // 标识结束
                exec(xargv[0], xargv);
            }
            else if (pid > 0)
            {
                // parent process
                wait(0);
                p = buf + i + 1; // p指向新的buf位置
            }
            else
            {
                // fork error
                fprintf(1, "fork error\n");
                exit(1);
            }
        }
    }
    exit(0);
}
```

```c
// xargs.c

#include "kernel/types.h"
#include "kernel/stat.h"
#include "user/user.h"

int main(int argc, char *argv[])
{
    char buf[1024]; // 存储从标准输入流读出字符串
    char *xargv[16];
    int xargc = 0;
    // 将argv的所有参数存储到xargv中，作为后续调用exec使用
    for (int i = 1; i < argc; i++)
        xargv[xargc++] = argv[i];
    char *p = buf; // p指向buf首地址
    char c;
    int cnt = 0;
    while (read(0, &c, sizeof(c)) > 0) // 一次读取一个字符,读多次
    {
        if (c == '\n')
        {
            int pid = fork();
            if (pid == 0)
            {
                // child process
                // 将buf的参数加到xargv中
                xargv[xargc++] = p;    // 进行连接
                xargv[xargc++] = '\0'; // 标识结束
                exec(xargv[0], xargv);
            }
            else if (pid > 0)
            {
                // parent process
                wait(0);
                p = buf + cnt + 1; // p指向新的buf位置
            }
            else
            {
                // fork error
                fprintf(1, "fork error\n");
                exit(1);
            }
        }
        else
        {
            buf[cnt++] = c; // 将c保存到buf数组
        }
    }
    exit(0);
}
```



![image-20221111154955231](http://shixiaozhong.oss-cn-hangzhou.aliyuncs.com/img/image-20221111154955231.png)

---

#### 总结

写一个对这个实验的总结吧，这个lab是最开始的一个lab，居然可以自己来实现shell下的命令，当时觉得非常的酷。看完了课程要求的资料就立刻开始了实验，第一个sleep是非常简单的，轻松拿下了，也给了我很大的自信。接下来的pingpong也没有很大的问题，因为开始忘了管道的内容，去网上找了几篇博客看了一下，就做出来了，也非常高兴。但是开始了后面的primes，就有点懵了，看了那张图以后也知道思路了，但是怎么实现成了问题，刚开始是想的循环，创建一批子进程啥的，想着想着不太对劲了，怎么传递数据呢？管道只能父子进程啊。兄弟进程怎么实现呢？然后就懵了，最后去看了一下网上提供的思路用递归才知道，这个应该不难想到啊，可能是太久没刷算法题了吧。后来继续find，这个find花了我最多的时间了，思路是不是很难的，但是写代码加调试，搞了很久才出来，还是因为递归的不熟练导致的，所以一定算法不能停。最后就是这个xargs了，这个其实是不难的，但是太菜了，也花费了不少时间，只需要简单的提取数据就可以。总时长记不清了，断断续续做的，估计得有十几个小时吧，主要写c语言的功底太差了。

最后执行`make grade`命令来完成总的测试，还需要创建一个time.txt文件，记录自己在这个lab上花费的时间。否则就只有99分。

![image-20221113222405352](http://shixiaozhong.oss-cn-hangzhou.aliyuncs.com/img/image-20221113222405352.png)

创建一个time.txt文件以后。

![image-20221111220932215](http://shixiaozhong.oss-cn-hangzhou.aliyuncs.com/img/image-20221111220932215.png)

