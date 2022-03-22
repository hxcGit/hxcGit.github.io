# SSH配置


<!--more-->

### 一、 远程用户：免密登陆+IP 重命名

- 用户目录下的`.ssh`目录内，添加配置文件`config`

```
Host 47.103.81.142
	HostName cloudHost
	User root
	ServerAliveInterval 20
```

`C:\Windows\System32\drivers\etc\hosts`需要添加内容

```
47.103.81.142 cloudHost
```

```
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
```

### 二、 连接不中断

- 修改配置文件`~/.ssh/config`

- 添加配置项`ServerAliveInterval=30`

