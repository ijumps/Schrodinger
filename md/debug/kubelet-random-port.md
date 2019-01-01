# Debug for go program (kubelet open random port)

## 缘起

发现某个版本开始，`kubelet` 每次运行后，会启动一个随机的 TCP 端口，绑定到 `127.0.0.1` 的本地端口上，可以通过 http 方式访问，返回都是 `404`，比如像下面这样的：

```
[root@dev test]# ss -lptn | grep kubelet
LISTEN     0      128    127.0.0.1:10248                    *:*                   users:(("kubelet",pid=5854,fd=20))
LISTEN     0      128    127.0.0.1:37231                    *:*                   users:(("kubelet",pid=5854,fd=7))
LISTEN     0      128         :::10250                   :::*                   users:(("kubelet",pid=5854,fd=17))
LISTEN     0      128         :::10255                   :::*                   users:(("kubelet",pid=5854,fd=19))
[root@dev test]# curl 127.0.0.1:37231/hi
404: Page Not Found
```

非固定端口的形式十分诡异，所以想要弄明白怎么回事。

## 破案

### google first

尝试以 `kubelet` `kubernetes` `k8s` `random port` `127.0.0.1` `localhost` 等关键字搜索，好像没什么相关内容

### upstream then

去翻了下社区 [issue](https://github.com/kubernetes/kubernetes/issues) ，以上面关键字搜索，好像也没什么相关内容

### how to

既然不是已知问题，那么就尝试从现象来找线索

**对象**: `kubelet` 进程，是一个 golang 编写的程序
**行为**: 监听了**本地随机**端口，协议是 **tcp**

**线索**:

1. `随机` 代表在 listen 时，1. 指定了一个随机端口；2. 指定的端口是 `0`
2. `golang` go 程序里面的监听某个端口的行为，通常是调用的 `net.Listen()` 方法

### next

通常，debug 到这一步时，`next` 就没有固定套路了。

比如，尝试搜索 `kubernetes` 源码里面，所有涉及 `net.Listen()` 调用的地方，筛选可能的配置端口是 `0` 的地方，发现了一些有趣的测试代码包含 `net.Listen("tcp", "127.0.0.1:0")` 这类随机端口，是用来测试的

### more

通常，debug 到这个节点，会有些不管正确还是错误的猜测，需要更多的信息来验证和排查。这个时候，可以人造些线索，比如：

1. 调高 `kubelet` 日志等级，看看能否提供更多的信息
2. 访问一个固定的 path ，比如 `127.0.0.1:37231/hahaha`，看看日志里面是否会出现
3. 尝试通过 `strace` 来看看发生了啥（单 `strace` 是 syscall 级别的，信息不一定有用；比如能看到 `hahaha` 关键字了，但是在 `read` `write` 的 syscall 里，还是不知道发生了什么）

### again

到这里，看来常规途径都没啥好的线索，那么还是回到最初的**确定**的现象

> 发现某个版本开始，`kubelet` 每次运行后，会启动一个随机的 TCP 端口，绑定到 `127.0.0.1` 的本地端口上，可以通过 http 方式访问，返回都是 `404`

既然上面用 `strace` 或者搜索源码都不太好找到线索，那么可以换个思路，或是换个工具

比如这里幻想着一个工具：

1. 可以调试运行中的软件
2. 可以在指定的地方触发
3. 可以打印运行的参数/内存

这么想想 ，好像 `gdb` 是做这么一件事的利器。但是，`gdb` 好用么？？？

### Delve

曾经用 `gdb` 调试过 `kubelet`，因为默认 release 版本的 `kubelet` 缺少调试的符号表，使用起来太过困难。

依旧 google 一波，关键字 `golang debug`，发现官方的 [Debugging Go Code with GDB](https://golang.org/doc/gdb) 上面推荐了一个工具，叫做 [Delve](https://github.com/derekparker/delve)

Delve 的安装和使用文档就不多说，参考官方文档就好 

尝试了一波，确实方便和强大，但还有最后一个问题，**断点打在哪里呢**？需要在监听这个端口的调用那里来打一个断点，来确认是这个调用监听的这个端口；但如果知道了这个调用在哪，我还打断点干啥。

如果解决`鸡生蛋蛋生鸡`的问题呢？不管鸡也不管蛋，直接断点到 `net.Listen` 这个调用

## Debug

（正文开始了）

找好了工具，确定了方法，下面 debug 过程就顺风顺水了

加入 `cmd/kubelet` 目录，debug 模式运行 kubelet

```
[root@dev kubelet]# cd ~/go/src/k8s.io/kubernetes/cmd/kubelet/
[root@dev kubelet]# dlv debug .
Type 'help' for list of commands.
(dlv)
```

设置断点

```
(dlv) break net.Listen
Breakpoint 1 set at 0x61da48 for net.Listen() /usr/local/go/src/net/dial.go:672
```

等待断点触发

```
(dlv) continue
...
> net.Listen() /usr/local/go/src/net/dial.go:672 (hits goroutine(1):1 total:2) (PC: 0x61da48)
> net.Listen() /usr/local/go/src/net/dial.go:672 (hits goroutine(98):1 total:2) (PC: 0x61da48)
   667: // The Addr method of Listener can be used to discover the chosen
   668: // port.
   669: //
   670: // See func Dial for a description of the network and address
   671: // parameters.
=> 672: func Listen(network, address string) (Listener, error) {
   673:         var lc ListenConfig
   674:         return lc.Listen(context.Background(), network, address)
   675: }
   676:
   677: // ListenPacket announces on the local network address.
(dlv)
```

打印 listen 信息

```
(dlv) p address
"localhost:0"
```

查看调用栈

```
(dlv) bt
0  0x000000000061da48 in net.Listen
   at /usr/local/go/src/net/dial.go:672
1  0x00000000025448f6 in k8s.io/kubernetes/pkg/kubelet/server/streaming.(*server).Start
   at /root/go/src/k8s.io/kubernetes/pkg/kubelet/server/streaming/server.go:238
2  0x00000000041b1a12 in k8s.io/kubernetes/pkg/kubelet/dockershim.(*dockerService).Start.func1
   at /root/go/src/k8s.io/kubernetes/pkg/kubelet/dockershim/docker_service.go:400
3  0x0000000000466451 in runtime.goexit
   at /usr/local/go/src/runtime/asm_amd64.s:1333
```

分步执行，并对比前后 `ss -lptn` 的变化

```
(dlv) n
> net.Listen() /usr/local/go/src/net/dial.go:674 (PC: 0x61da7e)
   669: //
   670: // See func Dial for a description of the network and address
   671: // parameters.
   672: func Listen(network, address string) (Listener, error) {
   673:         var lc ListenConfig
=> 674:         return lc.Listen(context.Background(), network, address)
   675: }
   676:
   677: // ListenPacket announces on the local network address.
   678: //
   679: // The network must be "udp", "udp4", "udp6", "unixgram", or an IP
(dlv)

---

[root@dev test]# ss -lptn
State       Recv-Q Send-Q     Local Address:Port                    Peer Address:Port
LISTEN      0      128                    *:22                                 *:*                   users:(("sshd",pid=3360,fd=3))
LISTEN      0      100            127.0.0.1:25                                 *:*                   users:(("master",pid=3728,fd=13))
LISTEN      0      128                   :::22                                :::*                   users:(("sshd",pid=3360,fd=4))
LISTEN      0      100                  ::1:25                                :::*                   users:(("master",pid=3728,fd=14))

---

(dlv) n
> k8s.io/kubernetes/pkg/kubelet/server/streaming.(*server).Start() /root/go/src/k8s.io/kubernetes/pkg/kubelet/server/streaming/server.go:238 (PC: 0x25448f6)
Values returned:
        ~r2: net.Listener(*net.TCPListener) *{
                fd: *net.netFD {
                        pfd: (*internal/poll.FD)(0xc00030fa80),
                        family: 2,
                        sotype: 1,
                        isConnected: false,
                        net: "tcp",
                        laddr: net.Addr(*net.TCPAddr) ...,
                        raddr: net.Addr nil,},}
        ~r3: error nil

   233:         if !stayUp {
   234:                 // TODO(tallclair): Implement this.
   235:                 return errors.New("stayUp=false is not yet implemented")
   236:         }
   237:
=> 238:         listener, err := net.Listen("tcp", s.config.Addr)
   239:         if err != nil {
   240:                 return err
   241:         }
   242:         // Use the actual address as baseURL host. This handles the "0" port case.
   243:         s.config.BaseURL.Host = listener.Addr().String()
(dlv)

---

[root@dev test]# ss -lptn
State       Recv-Q Send-Q     Local Address:Port                    Peer Address:Port
LISTEN      0      128                    *:22                                 *:*                   users:(("sshd",pid=3360,fd=3))
LISTEN      0      100            127.0.0.1:25                                 *:*                   users:(("master",pid=3728,fd=13))
LISTEN      0      128            127.0.0.1:33375                              *:*                   users:(("debug",pid=6847,fd=8))
LISTEN      0      128                   :::22                                :::*                   users:(("sshd",pid=3360,fd=4))
LISTEN      0      100                  ::1:25                                :::*                   users:(("master",pid=3728,fd=14))
```

到这里，就初步破案了，可以看到 `dockershim` 创建时，先启动了一个 `streaming server`，这个 **kubelet random port** 就是这个 `streaming server` 做的，初步看来是做 `exec attach portforward` 的

## 彩蛋

上面用 `dlv` 去跟踪断点时，并不是每次查看到的 listen 地址都是 `"localhost:0"`，比如多跑几次，可以看到

```
> net.Listen() /usr/local/go/src/net/dial.go:672 (hits goroutine(1):1 total:2) (PC: 0x61da48)
   667: // The Addr method of Listener can be used to discover the chosen
   668: // port.
   669: //
   670: // See func Dial for a description of the network and address
   671: // parameters.
=> 672: func Listen(network, address string) (Listener, error) {
   673:         var lc ListenConfig
   674:         return lc.Listen(context.Background(), network, address)
   675: }
   676:
   677: // ListenPacket announces on the local network address.
(dlv) bt
 0  0x000000000061da48 in net.Listen
    at /usr/local/go/src/net/dial.go:672
 1  0x00000000018e22b5 in k8s.io/kubernetes/pkg/kubelet/util.CreateListener
    at /root/go/src/k8s.io/kubernetes/pkg/kubelet/util/util_unix.go:53
 2  0x00000000041b35c1 in k8s.io/kubernetes/pkg/kubelet/dockershim/remote.(*DockerServer).Start
    at /root/go/src/k8s.io/kubernetes/pkg/kubelet/dockershim/remote/docker_server.go:60
 3  0x00000000042791a8 in k8s.io/kubernetes/pkg/kubelet.NewMainKubelet
    at /root/go/src/k8s.io/kubernetes/pkg/kubelet/kubelet.go:640
 4  0x00000000042dabd6 in k8s.io/kubernetes/cmd/kubelet/app.CreateAndInitKubelet
    at ./app/server.go:1100
 5  0x00000000042d9efa in k8s.io/kubernetes/cmd/kubelet/app.RunKubelet
    at ./app/server.go:990
 6  0x00000000042d46fc in k8s.io/kubernetes/cmd/kubelet/app.run
    at ./app/server.go:707
 7  0x00000000042d2cf3 in k8s.io/kubernetes/cmd/kubelet/app.Run
    at ./app/server.go:412
 8  0x00000000042ddc53 in k8s.io/kubernetes/cmd/kubelet/app.NewKubeletCommand.func1
    at ./app/server.go:261
 9  0x000000000409ad8a in k8s.io/kubernetes/vendor/github.com/spf13/cobra.(*Command).execute
    at /root/go/src/k8s.io/kubernetes/vendor/github.com/spf13/cobra/command.go:760
10  0x000000000409b72b in k8s.io/kubernetes/vendor/github.com/spf13/cobra.(*Command).ExecuteC
    at /root/go/src/k8s.io/kubernetes/vendor/github.com/spf13/cobra/command.go:846
11  0x000000000409b02b in k8s.io/kubernetes/vendor/github.com/spf13/cobra.(*Command).Execute
    at /root/go/src/k8s.io/kubernetes/vendor/github.com/spf13/cobra/command.go:794
12  0x00000000042dfde6 in main.main
    at ./kubelet.go:43
13  0x0000000000435bd5 in runtime.main
    at /usr/local/go/src/runtime/proc.go:201
14  0x0000000000466451 in runtime.goexit
    at /usr/local/go/src/runtime/asm_amd64.s:1333
```

这是因为 `goroutine` 的原因，可能同时有多个地方触发了这个断点（这里就不破案了）；但是可能和 `dlv` 实现有关，只能断点到其中一个 `goroutine` 中，另外一个 `goroutine` 会继续跑（埋个坑）

如果查看断点时的 `goroutine`，就可以看到

```
(dlv) goroutines
* Goroutine 1 - User: /usr/local/go/src/net/dial.go:672 net.Listen (0x61da48) (thread 7034)
  Goroutine 2 - User: /usr/local/go/src/runtime/proc.go:303 runtime.gopark (0x435fb4)
  Goroutine 3 - User: /usr/local/go/src/runtime/proc.go:303 runtime.gopark (0x435fb4)
  Goroutine 4 - User: /usr/local/go/src/runtime/lock_futex.go:228 runtime.notetsleepg (0x40ec57)
  Goroutine 18 - User: /usr/local/go/src/runtime/proc.go:303 runtime.gopark (0x435fb4)
  Goroutine 19 - User: /usr/local/go/src/runtime/proc.go:303 runtime.gopark (0x435fb4)
  Goroutine 20 - User: /usr/local/go/src/runtime/proc.go:303 runtime.gopark (0x435fb4)
  Goroutine 21 - User: /root/go/src/k8s.io/kubernetes/vendor/k8s.io/klog/klog.go:941 k8s.io/kubernetes/vendor/k8s.io/klog.(*loggingT).flushDaemon (0x94e6d3)
  Goroutine 33 - User: /usr/local/go/src/runtime/sigqueue.go:139 os/signal.signal_recv (0x44bedc)
  Goroutine 46 - User: /usr/local/go/src/runtime/proc.go:303 runtime.gopark (0x435fb4)
  Goroutine 47 - User: /root/go/src/k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server/signal.go:36 k8s.io/kubernetes/vendor/k8s.io/apiserver/pkg/server.SetupSignalHandler.func1 (0x1846e64)
  Goroutine 48 - User: /root/go/src/k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/wait/wait.go:145 k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/wait.JitterUntil (0x10b0f2a)
  Goroutine 63 - User: /usr/local/go/src/runtime/lock_futex.go:228 runtime.notetsleepg (0x40ec57)
  Goroutine 67 - User: /usr/local/go/src/runtime/netpoll.go:173 internal/poll.runtime_pollWait (0x4306ae)
  Goroutine 68 - User: /usr/local/go/src/net/http/transport.go:1885 net/http.(*persistConn).writeLoop (0x7ec84e)
  Goroutine 75 - User: /usr/local/go/src/runtime/netpoll.go:173 internal/poll.runtime_pollWait (0x4306ae)
  Goroutine 76 - User: /usr/local/go/src/net/http/transport.go:1885 net/http.(*persistConn).writeLoop (0x7ec84e)
  Goroutine 82 - User: /root/go/src/k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/watch/mux.go:207 k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/watch.(*Broadcaster).loop (0xb2c8e9)
  Goroutine 83 - User: /root/go/src/k8s.io/kubernetes/vendor/k8s.io/client-go/tools/record/event.go:231 k8s.io/kubernetes/vendor/k8s.io/client-go/tools/record.(*eventBroadcasterImpl).StartEventWatcher.func1 (0x18ea5d4)
  Goroutine 89 - User: /usr/local/go/src/net/dial.go:672 net.Listen (0x61da48) (thread 7016)
  Goroutine 90 - User: /root/go/src/k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/wait/wait.go:87 k8s.io/kubernetes/vendor/k8s.io/apimachinery/pkg/util/wait.Until (0x10b0d10)
[21 goroutines]
```

可以切换到我们需要 debug 的 `goroutine`，继续断点调试

```
(dlv) goroutine 89
Switched from 1 to 89 (thread 7016)
(dlv) bt
0  0x000000000061da48 in net.Listen
   at /usr/local/go/src/net/dial.go:672
1  0x00000000025448f6 in k8s.io/kubernetes/pkg/kubelet/server/streaming.(*server).Start
   at /root/go/src/k8s.io/kubernetes/pkg/kubelet/server/streaming/server.go:238
2  0x00000000041b1a12 in k8s.io/kubernetes/pkg/kubelet/dockershim.(*dockerService).Start.func1
   at /root/go/src/k8s.io/kubernetes/pkg/kubelet/dockershim/docker_service.go:400
3  0x0000000000466451 in runtime.goexit
   at /usr/local/go/src/runtime/asm_amd64.s:1333
```

## 后记

**埋个坑**，这段 `streaming server` 的逻辑还是比较诡异，也不知道是重构代码没删干净，还是真有在用；不知道有没有类似绕过 auth 的问题。。。
