# 如何给 kernel.org 下项目贡献代码

## 使用 git 提交代码

> git add your-file
> git commit -s -m "description of your commit"

-s 表示生成 Signed-off-by 的信息

## 使用 git 生成补丁

> git format-patch -1

这里会将最近一次的提交生成补丁文件，可以在当前目录下看到 0001-*.patch 文件

## 使用邮件发送你的 patch 给 maintainer 与 mailing list

代码库下会有文件描述 maintainer 的信息，这里以 rt-tests 为例，在 MAINTAINERS 文件中有 maintainer 的邮件地址，可将你的 patch 发给上述地址。

## 使用 Mutt 发送 patch

[email_clients](http://lxr.free-electrons.com/source/Documentation/email-clients.txt) 这里描述了许多可选用的 email 客户端，这里以 Mutt 为例：

使用 Mutt 必须配置好 msmtp，msmtp 的配置可从网络搜索，注意到 Gmail 在国内访问十分困难，这里转换思路来使用 Gmail。

在你的 git 客户端中需配置你的邮件地址为要发送的 Gmail 地址，配置方法如下：

> git config --global user.name "Yuanbin Zhou"
> git config --global user.email "hduffddybz@gmail.com"

而在 msmtp 中使用 126 邮箱来转发 gmail 邮件，msmtp的配置如下：

>account 126
>protocol smtp
>host smtp.126.com
>from ffddybz123@126.com
>user ffddybz123@126.com
>password *******
>auth on
>tls on
>tls_trust_file /etc/ssl/certs/ca-certificates.crt
>syslog LOG_MAIL

>account default:126

完成以上步骤后就可使用 mutt -H 0001-*.patch 来发送你的 patch，注意可在邮件中撰写收件者，抄送者，主题，以及简要的说明。

注意到这里未作 Mutt 的收信设置，其实你可以在其它邮件客户端或者网页版的 Gmail 中获取上游的反馈，而一旦需要更改 patch 需要重新写封邮件。

 

