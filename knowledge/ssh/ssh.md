# SSH 一些思考

## win

### windows配置 ssh 方便 AI 使用

1. 用 openssh 去配置 直接安装配置就好了
2. windows 管理员账号的配置信息在 c:/ProgramData/sshd 里面 包括 信任的私钥公钥对

## linux

1. Linux 需要明确好 ssh 的登录用户 如果是特定用户的 ssh 的话需要明确去对应的用户目录下创建 ssh 键
