---
layout: post
title:  " windows配置github ssh key笔记"
date:  2018-05-27 12:39:04
type: 笔记
categories: [笔记]
keywords: git,ssh,window,key
---
## 写在前面

最近电脑重装了一次，好多环境要重新安装，需要重新配置git ssh认证，以前配置是找的教程，现在配置还是找教程，浪费了不少时间，这次不如把整个过程记录下来。方便下次安装

### ssh
>什么是ssh：ssh是Secure Shell（安全外壳协议）的缩写，建立在应用层和传输层基础上的安全协议。为了便于访问github，要生成ssh公钥，这样就不用每一次访问github都要输入用户名和密码。

### 适用范围

1）window环境git设置ssh key，linux环境同理；（应该更简单）
2）已经安装好git
3）有git账号


## 配置步骤

1）使用 `ssh-keygen` 生成密钥
进入git 安装的bin目录，执行`bash`（D:\Program Files\Git\bin\bash）,邮箱为git注册邮箱（三次提示可以回车跳过）
```
ssh-keygen -t rsa -C "email@email.com"
```
2）git设置develop key（这里以github为例）

可以设置项目级别的ssh key，也可以设置为全局的，这里是家里的个人电脑，直接设置为全局的。
setting\SSH and GPG keys
到~/.ssh(C:\Users\Administrator\.ssh)目录，找到id_rsa.pub文件，拷贝内容添加到git ssh key里面。如图：
![Alt text](./images/1527389183912.png)
3）测试配置

```
ssh -T git@github.com
```

如果出现以下内容，说明配置成功

如果出现以下内容即表示配置完成并且成功！
![Alt text](./images/1527389450563.png)

```
Hi username! You've successfully authenticated, but GitHub does not
provide shell access.
```

## 参考文档
https://blog.csdn.net/hhgggggg/article/details/77853665
