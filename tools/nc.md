## 一、端口与连接
```bash
nc -v  host port               # 单端口连通性测试
nc -z -w 3 host 22-80          # 批量扫端口（仅 SYN/Connect）
nc -u host 53                  # UDP 模式
nc -v6 host port               # IPv6
```
## 二、临时服务器/文件传输
```bash
nc -l -p 8080                  # 裸监听 8080（传统版）
ncat -l 8080                   # ncat 版（可省略 -p）
nc -l -p 8080 > recv.file      # 收文件
nc host 8080 < send.file       # 发文件
tar czf - dir | nc -l -p 9999  # 边打包边传
nc host 9999 | tar xzf -       # 对端边收边解
```
## 三、执行壳（Classic Backdoor，仅传统版）
```bash
nc -l -p 4444 -e /bin/bash     # 服务器端给 shell
nc -l -p 4444 -e cmd.exe       # Windows 同理
```
> ncat 用 `--exec` 或 `--sh-exec`；多数发行版缺省编译**去掉 `-e`**，需自行 rebuild。
## 四、高级加密/代理（ncat 专属）
```bash
ncat -l 9999 --ssl             # SSL 监听
ncat host 9999 --ssl           # SSL 客户端
ncat -l 8080 --proxy-type http --proxy 127.0.0.1:3128  # 前端代理
```
## 五、超时与保活
```bash
nc -w 5 host port              # 连接/闲置超时 5 s
nc -i 1 host port              # 每发一行停顿 1 s（传统版）
nc -l -p 9999 -k              # 长连接模式（1 次 accept 不退）
ncat -l 9999 --keep-open       # ncat 等价
```
## 六、源地址/速率/调试
```bash
nc -s 192.168.1.100 host port  # 指定本地源 IP
nc -v -v host port             # 双 -v 更啰嗦
nc -o hex.log host port        # ncat：十六进制转储
nc -d -d host port             # 传统版：调试级别 2
```
## 七、最常用 5 句背走
```bash
nc -v  ip port                 # 通不通
nc -z -w3 ip 22-80            # 扫端口
nc -l -p 9999                  # 临时服务器
nc -l -p 9999 > f              # 收文件
tar czf - dir | nc -l -p 9999  # 裸机备份
```
