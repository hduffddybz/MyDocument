# ��θ� kernel.org ����Ŀ���״���

## ʹ�� git �ύ����

> git add your-file
> git commit -s -m "description of your commit"

-s ��ʾ���� Signed-off-by ����Ϣ

## ʹ�� git ���ɲ���

> git format-patch -1

����Ὣ���һ�ε��ύ���ɲ����ļ��������ڵ�ǰĿ¼�¿��� 0001-*.patch �ļ�

## ʹ���ʼ�������� patch �� maintainer �� mailing list

������»����ļ����� maintainer ����Ϣ�������� rt-tests Ϊ������ MAINTAINERS �ļ����� maintainer ���ʼ���ַ���ɽ���� patch ����������ַ��

## ʹ�� Mutt ���� patch

[email_clients](http://lxr.free-electrons.com/source/Documentation/email-clients.txt) ��������������ѡ�õ� email �ͻ��ˣ������� Mutt Ϊ����

ʹ�� Mutt �������ú� msmtp��msmtp �����ÿɴ�����������ע�⵽ Gmail �ڹ��ڷ���ʮ�����ѣ�����ת��˼·��ʹ�� Gmail��

����� git �ͻ���������������ʼ���ַΪҪ���͵� Gmail ��ַ�����÷������£�

> git config --global user.name "Yuanbin Zhou"
> git config --global user.email "hduffddybz@gmail.com"

���� msmtp ��ʹ�� 126 ������ת�� gmail �ʼ���msmtp���������£�

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

������ϲ����Ϳ�ʹ�� mutt -H 0001-*.patch ��������� patch��ע������ʼ���׫д�ռ��ߣ������ߣ����⣬�Լ���Ҫ��˵����

ע�⵽����δ�� Mutt ���������ã���ʵ������������ʼ��ͻ��˻�����ҳ��� Gmail �л�ȡ���εķ�������һ����Ҫ���� patch ��Ҫ����д���ʼ���

 

