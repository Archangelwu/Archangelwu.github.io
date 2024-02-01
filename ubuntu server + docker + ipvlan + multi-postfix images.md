# ubuntu server + postfix + postmulti + OpenDKIM + TLS

目的：在一台 ubuntu server 服務器上使用多個虛擬 IP 並服務多個 postfix multi-instances，形成在一台服務器上，有多個且獨立的 smtp 分別寄送郵件。

:::info
在此預設你會：
- CCNA 相關知識。
    - 熟悉防火牆設定。
    - 熟悉 Core Switch 設定。
    - 熟悉 TCP/IP OSI 7 layer。
- Esxi vCenter 設定。
:::

### 以下開始看 shell 命令並稍加說明：
```shell
❯ ssh -i .ssh/id_rsa_for_smtp victor-postfix@10.1.11.43
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-91-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Thu Feb  1 10:17:51 AM CST 2024

  System load:  0.0                Processes:               152
  Usage of /:   40.5% of 14.66GB   Users logged in:         0
  Memory usage: 6%                 IPv4 address for ens160: 10.1.11.43
  Swap usage:   0%                 IPv4 address for ens160: 10.1.11.45


Expanded Security Maintenance for Applications is not enabled.

47 updates can be applied immediately.
To see these additional updates run: apt list --upgradable

Enable ESM Apps to receive additional future security updates.
See https://ubuntu.com/esm or run: sudo pro status


The list of available updates is more than a week old.
To check for new updates run: sudo apt update
Failed to connect to https://changelogs.ubuntu.com/meta-release-lts. Check your Internet connection or proxy settings


Last login: Wed Jan 31 14:33:20 2024 from 10.0.8.245
```
已在 esxi 上安裝 Ubuntu 22.04.3 LTS，並且 mapping 多個內部 IP，如果需要看網路設定，請參閱詳細資訊，在此就不贅述。

:::spoiler
```shell
victor-postfix@victorpostfix:~$ cat /etc/netplan/00-installer-config.yaml
# This is the network config written by 'subiquity'
network:
  ethernets:
    ens160:
      addresses:
      - 10.1.11.43/24
      - 10.1.11.45/24
      nameservers:
        addresses:
        - 10.1.0.72
        - 10.1.0.73
        search: []
      routes:
      - to: default
        via: 10.1.11.254
  version: 2
```
設定完畢後記得要讓其生效：
```shell
sudo netplan apply
```

:::

### 驗證網路配置

```shell
victor-postfix@victorpostfix:~$ ip addr show ens160
2: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:50:56:a5:c5:e6 brd ff:ff:ff:ff:ff:ff
    altname enp3s0
    inet 10.1.11.43/24 brd 10.1.11.255 scope global ens160
       valid_lft forever preferred_lft forever
    inet 10.1.11.45/24 brd 10.1.11.255 scope global secondary ens160
       valid_lft forever preferred_lft forever
    inet6 fe80::250:56ff:fea5:c5e6/64 scope link
       valid_lft forever preferred_lft forever
```

### 安裝 Postfix

```shell
sudo apt-get update
```
更新套件庫。

```shell
sudo apt-get install postfix
```
安裝 postfix

### 安裝 OpenDKIM

```shell
sudo apt-get install opendkim opendkim-tools
```

### 設置啟用 Postmulti

```shell
sudo nano /etc/postfix/main.cf

mynetworks_style = subnet
mynetworks = 10.0.0.0/16, 10.1.0.0/16

mailbox_size_limit = 0
recipient_delimiter = +
inet_interfaces = localhost
inet_protocols = all

multi_instance_enable = yes
multi_instance_wrapper = ${command_directory}/postmulti -p --

debug_peer_level = 2
```
主要是添加這一行

:::warning
==**multi_instance_enable = yes**==
:::



### 初始化 Postmulti

```shell
sudo postmulti -e init
```

### 為每個 IP 建立 postfix instance

```shell=
sudo postmulti -I postfix-43 -e create
sudo postmulti -I postfix-45 -e create
```
每個 postfix instance 都會有自己的配置文件，位於 /etc/postfix-(instance_number)

### 編輯 postfix-43/main.cf 與 postfix-45/main.cf

主要是添加以下幾行：
#### postfix-43/main.cf
:::warning
==**multi_instance_name = postfix-43**==

==**multi_instance_group = mail**==

==**inet_interfaces = 10.1.11.43**==

==**multi_instance_enable = yes**==
:::
#### postfix-45/main.cf
:::warning
==**multi_instance_name = postfix-45**==

==**multi_instance_group = mail**==

==**inet_interfaces = 10.1.11.45**==

==**multi_instance_enable = yes**==
:::

### 重新啟動 postfix instances

```shell=
sudo postmulti -i postfix-43 -p restart
sudo postmulti -i postfix-45 -p restart
```

### 檢查 postfix instances status

```shell=
sudo postmulti -i postfix-43 -p status
sudo postmulti -i postfix-45 -p status
```

### 檢查 instances 狀態

```shell=
victor-postfix@victorpostfix:~$ sudo postmulti -l
-               -               y         /etc/postfix
postfix-43      mail            n         /etc/postfix-43
postfix-45      mail            n         /etc/postfix-45
postfix-46      -               n         /etc/postfix-46
```
這裡的設定已經是啟用了自動啟動功能，所以看到 y 的參數，沒啟用的是顯示 n 的參數。
而 postfix-43 mail 的意思是指 postfix-43 與 postfix-45 都在 mail 的 Group 裡頭！

### 讓 postfix instances 啟用自動啟動功能

```shell=
sudo postmulti -i postfix-43 -e enable
sudo postmulti -i postfix-45 -e enable
```
```shell=
victor-postfix@victorpostfix:~$ sudo postmulti -l
-               -               y         /etc/postfix
postfix-43      mail            y         /etc/postfix-43
postfix-45      mail            y         /etc/postfix-45
postfix-46      -               n         /etc/postfix-46
```

### ==Group 查詢==

```shell=
victor-postfix@victorpostfix:~$ sudo postmulti -g mail -p status
postfix-43/postfix-script: the Postfix mail system is running: PID: 1584
postfix-45/postfix-script: the Postfix mail system is running: PID: 1585
```

### 測試

```shell
 telnet 10.1.11.43 25
Trying 10.1.11.43...
Connected to 10.1.11.43.
Escape character is '^]'.
220 s43.11.ecrm.com.tw ESMTP Postfix (Ubuntu)
^C
quit

telnet 10.1.11.45 25
Trying 10.1.11.45...
Connected to 10.1.11.45.
Escape character is '^]'.
220 s45.11.ecrm.com.tw ESMTP Postfix (Ubuntu)
quit
221 2.0.0 Bye
```
測試成功。


## OpenDKIM
### 安裝 Opendkim

```shell
sudo apt-get update
sudo apt-get install opendkim opendkim-tools
```

### 設定 Opendkim

```shell
sudo nano /etc/opendkim.conf

KeyFile /etc/opendkim/keys/default.private
Selector default
KeyTable /etc/opendkim/KeyTable
SigningTable /etc/opendkim/SigningTable
Socket  inet:8891@localhost
InternalHosts  10.1.0.0/16, 10.0.0.0/16
```

### 建立加解密鑰的目錄

```shell
sudo mkdir -p /etc/opendkim/keys
```

### 在每個 postfix instances main.cf 加上：

```shell
#DKIM Parameters
smtpd_milters           = inet:127.0.0.1:8891
non_smtpd_milters       = $smtpd_milters
milter_default_action   = accept
milter_protocol         = 2
```
參數說明：
:::spoiler
- smtpd_milters = inet:127.0.0.1:8891：
    - 這個設定是告訴 Postfix 的 SMTP 服務要連接到哪個郵件過濾器（milter）。這裡設定為在本機（127.0.0.1）的 8891 端口。
    - 通常這個過濾器是用來進行 DKIM 簽名或其他類型的郵件處理。
    - 就是說，Postfix 會把所有要發送的郵件先送到這個端口的服務處理，比如加上 DKIM 數位簽名。
- non_smtpd_milters = $smtpd_milters：
    - 這個設定是用來指定非 SMTP 進程（例如本地郵件投遞或郵件隊列處理）使用的 milter 服務。
    - 設定為 $smtpd_milters，這表示非 SMTP 進程會使用跟 SMTP 進程相同的 milter 設定，確保所有的郵件處理都一致。
- milter_default_action = accept：
    - 這個設定定義了當無法連接到 milter 服務或服務出錯時，Postfix 應該怎麼辦。
    - 設定為 accept，意味著即使 milter 服務無回應或出錯，Postfix 還是會接受並處理郵件。這樣可以確保在 milter 服務有問題時，不會錯過或拒絕合法郵件。
- milter_protocol = 2：
    - 這個設定指的是 Postfix 跟 milter 之間通訊所用的協定版本。
    - 2 代表使用 milter 協定的第二版本，這是一個支援較新功能的版本，大部分現代的 milter 應用都兼容這個版本。

這些設定通常用於配置 DKIM（一種郵件驗證機制）或用於檢查郵件內容的其他過濾器。透過這些設定，Postfix 能夠在發送郵件過程中添加數字簽名，或進行其他形式的郵件處理。
:::

之後要重啟 instances 
```shell
sudo postmulti -g mail -p restart
```

### 重啟 Opendkim

```shell
sudo systemctl restart opendkim
```

### 查看詳細日誌

```shell
journalctl -xeu opendkim.service
```

```shell
victor-postfix@victorpostfix:~$ journalctl -xeu opendkim.service
░░ Subject: Automatic restarting of a unit has been scheduled
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░
░░ Automatic restarting of the unit opendkim.service has been scheduled, as the result for
░░ the configured Restart= setting for the unit.
Jan 08 08:46:42 victorpostfix systemd[1]: Stopped OpenDKIM Milter.
░░ Subject: A stop job for unit opendkim.service has finished
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░
░░ A stop job for unit opendkim.service has finished.
░░
░░ The job identifier is 6380 and the job result is done.
Jan 08 08:46:42 victorpostfix systemd[1]: opendkim.service: Start request repeated too quickly.
Jan 08 08:46:42 victorpostfix systemd[1]: opendkim.service: Failed with result 'exit-code'.
░░ Subject: Unit failed
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░
░░ The unit opendkim.service has entered the 'failed' state with result 'exit-code'.
Jan 08 08:46:42 victorpostfix systemd[1]: Failed to start OpenDKIM Milter.
░░ Subject: A start job for unit opendkim.service has failed
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░
░░ A start job for unit opendkim.service has finished with a failure.
░░
░░ The job identifier is 6380 and the job result is failed.
```

### 錯誤排解-1 ～

#### 建立與設置 KeyTable 和 SigningTable

```shell
sudo touch /etc/opendkim/KeyTable
sudo touch /etc/opendkim/SigningTable
```
#### 權限設置

```shell
sudo chown opendkim:opendkim /etc/opendkim/KeyTable
sudo chown opendkim:opendkim /etc/opendkim/SigningTable
sudo chmod 640 /etc/opendkim/KeyTable
sudo chmod 640 /etc/opendkim/SigningTable
```

#### 重啟 opendkim

```shell
sudo systemctl restart opendkim
```

再次檢查

```shell
victor-postfix@victorpostfix:/etc$ journalctl -xeu opendkim.service
░░ Support: http://www.ubuntu.com/support
░░
░░ The unit opendkim.service has successfully entered the 'dead' state.
Jan 08 09:18:19 victorpostfix systemd[1]: Stopped OpenDKIM Milter.
░░ Subject: A stop job for unit opendkim.service has finished
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░
░░ A stop job for unit opendkim.service has finished.
░░
░░ The job identifier is 6552 and the job result is done.
Jan 08 09:18:19 victorpostfix systemd[1]: Starting OpenDKIM Milter...
░░ Subject: A start job for unit opendkim.service has begun execution
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░
░░ A start job for unit opendkim.service has begun execution.
░░
░░ The job identifier is 6552.
Jan 08 09:18:19 victorpostfix systemd[1]: opendkim.service: Can't open PID file /run/opendkim/opendkim.pid (yet?) after start: Operation not permitted
Jan 08 09:18:19 victorpostfix systemd[1]: Started OpenDKIM Milter.
░░ Subject: A start job for unit opendkim.service has finished successfully
░░ Defined-By: systemd
░░ Support: http://www.ubuntu.com/support
░░
░░ A start job for unit opendkim.service has finished successfully.
░░
░░ The job identifier is 6552.
Jan 08 09:18:19 victorpostfix opendkim[16851]: OpenDKIM Filter v2.11.0 starting
```

### 錯誤排解-2 ～

```shell
victor-postfix@victorpostfix:~$ cat /lib/systemd/system/opendkim.service
[Unit]
Description=OpenDKIM Milter
Documentation=man:opendkim(8) man:opendkim.conf(5) man:opendkim-lua(3) man:opendkim-genkey(8) man:opendkim-genzone(8) man:opendkim-testkey(8) http://www.opendkim.org/docs.html
After=network.target nss-lookup.target

[Service]
Type=forking
#PIDFile=/run/opendkim/opendkim.pid
ExecStart=/usr/sbin/opendkim
ExecReload=/bin/kill -USR1 $MAINPID
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
:::danger
請將這一行註解掉...

#PIDFile=/run/opendkim/opendkim.pid

同樣的在 /etc/opendkim.conf 裡的設定一樣要註解掉

#PidFile			/run/opendkim/opendkim.pid
:::

最後 ..... 終於 .....

```shell
victor-postfix@victorpostfix:/etc$ sudo systemctl status opendkim
● opendkim.service - OpenDKIM Milter
     Loaded: loaded (/lib/systemd/system/opendkim.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2024-01-08 18:14:44 CST; 13s ago
       Docs: man:opendkim(8)
             man:opendkim.conf(5)
             man:opendkim-lua(3)
             man:opendkim-genkey(8)
             man:opendkim-genzone(8)
             man:opendkim-testkey(8)
             http://www.opendkim.org/docs.html
    Process: 17356 ExecStart=/usr/sbin/opendkim (code=exited, status=0/SUCCESS)
   Main PID: 17357 (opendkim)
      Tasks: 6 (limit: 4558)
     Memory: 1.9M
        CPU: 24ms
     CGroup: /system.slice/opendkim.service
             └─17357 /usr/sbin/opendkim

Jan 08 18:14:44 victorpostfix systemd[1]: Stopped OpenDKIM Milter.
Jan 08 18:14:44 victorpostfix systemd[1]: Starting OpenDKIM Milter...
Jan 08 18:14:44 victorpostfix systemd[1]: Started OpenDKIM Milter.
Jan 08 18:14:44 victorpostfix opendkim[17357]: OpenDKIM Filter v2.11.0 starting
```

## 設置 TLS

:::info
由於設置就於簡單，原理與 SSL 觀念相同，所以直接貼上設置結果，重點處也會稍微說明。
:::

### 修改 postfix instances master.cf

```shell
submission inet n       -       y       -       -       smtpd
  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes
#  -o smtpd_tls_auth_only=yes
  -o smtpd_reject_unlisted_recipient=no
  -o smtpd_client_restrictions=permit_sasl_authenticated,reject
#  -o smtpd_helo_restrictions=$mua_helo_restrictions
#  -o smtpd_sender_restrictions=$mua_sender_restrictions
#  -o smtpd_recipient_restrictions=
#  -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
  -o milter_macro_daemon_name=ORIGINATING
# Choose one: enable smtps for loopback clients only, or for any client.
#127.0.0.1:smtps inet n  -       y       -       -       smtpd
smtps     inet  n       -       y       -       -       smtpd
  -o syslog_name=postfix/smtps
  -o smtpd_tls_wrappermode=yes
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_reject_unlisted_recipient=no
  -o smtpd_client_restrictions=permit_sasl_authenticated,reject
#  -o smtpd_helo_restrictions=$mua_helo_restrictions
#  -o smtpd_sender_restrictions=$mua_sender_restrictions
#  -o smtpd_recipient_restrictions=
#  -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
  -o milter_macro_daemon_name=ORIGINATING
```

### 修改 postfix instances main.cf

```shell
# TLS
smtpd_tls_security_level = may
smtp_tls_security_level = may
# smtp_tls_security_level = encrypt
smtpd_tls_cert_file = /home/victor-postfix/ssl/2023-02-14-wildcard-ecrm-com-tw.pem
smtpd_tls_key_file = /home/victor-postfix/ssl/2023-02-14-wildcard-ecrm-com-tw.key
smtpd_tls_CAfile = /home/victor-postfix/ssl/alphasslrootcabundle-g4.crt
smtpd_use_tls=yes
smtp_tls_loglevel = 1
smtpd_tls_loglevel = 1
```

### 展示目錄權限與檔案權限

```shell
victor-postfix@victorpostfix:~$ pwd
/home/victor-postfix
victor-postfix@victorpostfix:~$ ll
total 64
drwxr-x--- 6 victor-postfix victor-postfix 4096 Feb  1 12:21 ./
drwxr-xr-x 3 root           root           4096 Jan  8 10:13 ../
-rw------- 1 victor-postfix victor-postfix 2237 Feb  1 12:55 .bash_history
-rw-r--r-- 1 victor-postfix victor-postfix  220 Jan  7  2022 .bash_logout
-rw-r--r-- 1 victor-postfix victor-postfix 3771 Jan  7  2022 .bashrc
drwx------ 2 victor-postfix victor-postfix 4096 Jan  8 10:14 .cache/
-rw------- 1 victor-postfix victor-postfix   20 Feb  1 12:21 .lesshst
drwxrwxr-x 3 victor-postfix victor-postfix 4096 Feb  1 10:39 .local/
-rw-r--r-- 1 victor-postfix victor-postfix  807 Jan  7  2022 .profile
-rw------- 1 victor-postfix victor-postfix  891 Jan 10 12:30 selector1-tt5
-rw------- 1 victor-postfix victor-postfix  331 Jan 10 12:30 selector1-tt5.txt
drwx------ 2 victor-postfix victor-postfix 4096 Jan  8 10:14 .ssh/
drwxr-x--- 2 victor-postfix victor-postfix 4096 Jan 29 15:33 ssl/
-rw-r--r-- 1 victor-postfix victor-postfix    0 Jan  8 10:35 .sudo_as_admin_successful
-rw------- 1 victor-postfix victor-postfix 9684 Jan 29 15:19 .viminfo
victor-postfix@victorpostfix:~$ ll ssl
total 28
drwxr-x--- 2 victor-postfix victor-postfix 4096 Jan 29 15:33 ./
drwxr-x--- 6 victor-postfix victor-postfix 4096 Feb  1 12:21 ../
-r-------- 1 victor-postfix victor-postfix 1675 Jan 29 15:33 2023-02-14-wildcard-ecrm-com-tw.key
-r--r--r-- 1 victor-postfix victor-postfix 4374 Jan 29 15:33 2023-02-14-wildcard-ecrm-com-tw.pem
-rw-r----- 1 victor-postfix victor-postfix 3485 Jan 29 15:33 2023-02-14-wildcard-ecrm-com-tw.pfx
-r--r--r-- 1 victor-postfix victor-postfix 2942 Jan 29 15:33 alphasslrootcabundle-g4.crt
```

PS:
pfx、pem、key、crt 等相關之間的轉換，不再贅述啦！

## 附註
:::info
以下附上各個設定好的檔案內容備份。
:::

### /etc/postfix/main.cf
:::spoiler
```shell=
victor-postfix@victorpostfix:~$ cat /etc/postfix/main.cf
# See /usr/share/postfix/main.cf.dist for a commented, more complete version


# Debian specific:  Specifying a file name will cause the first
# line of that file to be used as the name.  The Debian default
# is /etc/mailname.
#myorigin = /etc/mailname

smtpd_banner = $myhostname ESMTP $mail_name (Ubuntu)
biff = no

mail_name = WebMAX

# appending .domain is the MUA's job.
append_dot_mydomain = no

# Uncomment the next line to generate "delayed mail" warnings
#delay_warning_time = 4h

readme_directory = no

# See http://www.postfix.org/COMPATIBILITY_README.html -- default to 3.6 on
# fresh installs.
compatibility_level = 3.6



# TLS parameters
smtpd_tls_cert_file=/etc/ssl/certs/ssl-cert-snakeoil.pem
smtpd_tls_key_file=/etc/ssl/private/ssl-cert-snakeoil.key
smtpd_tls_security_level=may

smtp_tls_CApath=/etc/ssl/certs
smtp_tls_security_level=may
smtp_tls_session_cache_database = btree:${data_directory}/smtp_scache


smtpd_relay_restrictions = permit_mynetworks permit_sasl_authenticated defer_unauth_destination
myhostname = postfix-main-server.ecrm.com.tw
alias_maps = hash:/etc/aliases
alias_database = hash:/etc/aliases
myorigin = /etc/mailname
mydestination = $myhostname, victorpostfix.com, victorpostfix, localhost.localdomain, localhost
relayhost =
# mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128
mynetworks_style = subnet
mynetworks = 10.0.0.0/16, 10.1.0.0/16

mailbox_size_limit = 0
recipient_delimiter = +
inet_interfaces = localhost
inet_protocols = all

multi_instance_enable = yes
multi_instance_wrapper = ${command_directory}/postmulti -p --
multi_instance_directories = /etc/postfix-43 /etc/postfix-45 /etc/postfix-46

debug_peer_level = 2
```
:::

### /etc/postfix/master.cf
:::spoiler
```shell=
victor-postfix@victorpostfix:~$ cat /etc/postfix/master.cf
#
# Postfix master process configuration file.  For details on the format
# of the file, see the master(5) manual page (command: "man 5 master" or
# on-line: http://www.postfix.org/master.5.html).
#
# Do not forget to execute "postfix reload" after editing this file.
#
# ==========================================================================
# service type  private unpriv  chroot  wakeup  maxproc command + args
#               (yes)   (yes)   (no)    (never) (100)
# ==========================================================================
smtp      inet  n       -       y       -       -       smtpd
#smtp      inet  n       -       y       -       1       postscreen
#smtpd     pass  -       -       y       -       -       smtpd
#dnsblog   unix  -       -       y       -       0       dnsblog
#tlsproxy  unix  -       -       y       -       0       tlsproxy
# Choose one: enable submission for loopback clients only, or for any client.
#127.0.0.1:submission inet n -   y       -       -       smtpd
#submission inet n       -       y       -       -       smtpd
#  -o syslog_name=postfix/submission
#  -o smtpd_tls_security_level=encrypt
#  -o smtpd_sasl_auth_enable=yes
#  -o smtpd_tls_auth_only=yes
#  -o smtpd_reject_unlisted_recipient=no
#  -o smtpd_client_restrictions=$mua_client_restrictions
#  -o smtpd_helo_restrictions=$mua_helo_restrictions
#  -o smtpd_sender_restrictions=$mua_sender_restrictions
#  -o smtpd_recipient_restrictions=
#  -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
#  -o milter_macro_daemon_name=ORIGINATING
# Choose one: enable smtps for loopback clients only, or for any client.
#127.0.0.1:smtps inet n  -       y       -       -       smtpd
#smtps     inet  n       -       y       -       -       smtpd
#  -o syslog_name=postfix/smtps
#  -o smtpd_tls_wrappermode=yes
#  -o smtpd_sasl_auth_enable=yes
#  -o smtpd_reject_unlisted_recipient=no
#  -o smtpd_client_restrictions=$mua_client_restrictions
#  -o smtpd_helo_restrictions=$mua_helo_restrictions
#  -o smtpd_sender_restrictions=$mua_sender_restrictions
#  -o smtpd_recipient_restrictions=
#  -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
#  -o milter_macro_daemon_name=ORIGINATING
#628       inet  n       -       y       -       -       qmqpd
pickup    unix  n       -       y       60      1       pickup
cleanup   unix  n       -       y       -       0       cleanup
qmgr      unix  n       -       n       300     1       qmgr
#qmgr     unix  n       -       n       300     1       oqmgr
tlsmgr    unix  -       -       y       1000?   1       tlsmgr
rewrite   unix  -       -       y       -       -       trivial-rewrite
bounce    unix  -       -       y       -       0       bounce
defer     unix  -       -       y       -       0       bounce
trace     unix  -       -       y       -       0       bounce
verify    unix  -       -       y       -       1       verify
flush     unix  n       -       y       1000?   0       flush
proxymap  unix  -       -       n       -       -       proxymap
proxywrite unix -       -       n       -       1       proxymap
smtp      unix  -       -       y       -       -       smtp
relay     unix  -       -       y       -       -       smtp
        -o syslog_name=postfix/$service_name
#       -o smtp_helo_timeout=5 -o smtp_connect_timeout=5
showq     unix  n       -       y       -       -       showq
error     unix  -       -       y       -       -       error
retry     unix  -       -       y       -       -       error
discard   unix  -       -       y       -       -       discard
local     unix  -       n       n       -       -       local
virtual   unix  -       n       n       -       -       virtual
lmtp      unix  -       -       y       -       -       lmtp
anvil     unix  -       -       y       -       1       anvil
scache    unix  -       -       y       -       1       scache
postlog   unix-dgram n  -       n       -       1       postlogd
#
# ====================================================================
# Interfaces to non-Postfix software. Be sure to examine the manual
# pages of the non-Postfix software to find out what options it wants.
#
# Many of the following services use the Postfix pipe(8) delivery
# agent.  See the pipe(8) man page for information about ${recipient}
# and other message envelope options.
# ====================================================================
#
# maildrop. See the Postfix MAILDROP_README file for details.
# Also specify in main.cf: maildrop_destination_recipient_limit=1
#
maildrop  unix  -       n       n       -       -       pipe
  flags=DRXhu user=vmail argv=/usr/bin/maildrop -d ${recipient}
#
# ====================================================================
#
# Recent Cyrus versions can use the existing "lmtp" master.cf entry.
#
# Specify in cyrus.conf:
#   lmtp    cmd="lmtpd -a" listen="localhost:lmtp" proto=tcp4
#
# Specify in main.cf one or more of the following:
#  mailbox_transport = lmtp:inet:localhost
#  virtual_transport = lmtp:inet:localhost
#
# ====================================================================
#
# Cyrus 2.1.5 (Amos Gouaux)
# Also specify in main.cf: cyrus_destination_recipient_limit=1
#
#cyrus     unix  -       n       n       -       -       pipe
#  flags=DRX user=cyrus argv=/cyrus/bin/deliver -e -r ${sender} -m ${extension} ${user}
#
# ====================================================================
# Old example of delivery via Cyrus.
#
#old-cyrus unix  -       n       n       -       -       pipe
#  flags=R user=cyrus argv=/cyrus/bin/deliver -e -m ${extension} ${user}
#
# ====================================================================
#
# See the Postfix UUCP_README file for configuration details.
#
uucp      unix  -       n       n       -       -       pipe
  flags=Fqhu user=uucp argv=uux -r -n -z -a$sender - $nexthop!rmail ($recipient)
#
# Other external delivery methods.
#
ifmail    unix  -       n       n       -       -       pipe
  flags=F user=ftn argv=/usr/lib/ifmail/ifmail -r $nexthop ($recipient)
bsmtp     unix  -       n       n       -       -       pipe
  flags=Fq. user=bsmtp argv=/usr/lib/bsmtp/bsmtp -t$nexthop -f$sender $recipient
scalemail-backend unix -       n       n       -       2       pipe
  flags=R user=scalemail argv=/usr/lib/scalemail/bin/scalemail-store ${nexthop} ${user} ${extension}
mailman   unix  -       n       n       -       -       pipe
  flags=FRX user=list argv=/usr/lib/mailman/bin/postfix-to-mailman.py ${nexthop} ${user}
```
:::


### /etc/postfix-43/main.cf
:::spoiler
```shell=
victor-postfix@victorpostfix:~$ cat /etc/postfix-43/main.cf |grep -v '#'

compatibility_level = 3.6

data_directory = /var/lib/postfix-43

myhostname = s43.11.ecrm.com.tw
mydomain = $myhostname

unknown_local_recipient_reject_code = 550

mynetworks_style = subnet

mynetworks = 10.0.0.0/16, 10.1.0.0/16

alias_maps = hash:/etc/aliases

alias_database = hash:/etc/aliases

smtpd_banner = $myhostname ESMTP $mail_name (Ubuntu)

debug_peer_level = 2


debugger_command =
	 PATH=/bin:/usr/bin:/usr/local/bin:/usr/X11R6/bin
	 ddd $daemon_directory/$process_name $process_id & sleep 5

readme_directory = no
maximal_queue_lifetime = 6h
mail_name = WebMAX
smtp_destination_concurrency_limit = 2
bounce_size_limit = 10000000
maximal_backoff_time = 900
inet_protocols = ipv4
authorized_submit_users =
queue_directory = /var/spool/postfix-43
multi_instance_name = postfix-43
multi_instance_group = mail

inet_interfaces = 10.1.11.43
multi_instance_enable = yes

smtpd_milters           = inet:127.0.0.1:8891
non_smtpd_milters       = $smtpd_milters
milter_default_action   = accept
milter_protocol         = 2

smtpd_tls_security_level = may
smtp_tls_security_level = may
smtpd_tls_cert_file = /home/victor-postfix/ssl/2023-02-14-wildcard-ecrm-com-tw.pem
smtpd_tls_key_file = /home/victor-postfix/ssl/2023-02-14-wildcard-ecrm-com-tw.key
smtpd_tls_CAfile = /home/victor-postfix/ssl/alphasslrootcabundle-g4.crt
smtpd_use_tls=yes
smtp_tls_loglevel = 1
smtpd_tls_loglevel = 1
```
:::


### /etc/postfix-43/master.cf
:::spoiler
```shell=
victor-postfix@victorpostfix:~$ cat /etc/postfix-43/master.cf
#
# Postfix master process configuration file.  For details on the format
# of the file, see the master(5) manual page (command: "man 5 master" or
# on-line: http://www.postfix.org/master.5.html).
#
# Do not forget to execute "postfix reload" after editing this file.
#
# ==========================================================================
# service type  private unpriv  chroot  wakeup  maxproc command + args
#               (yes)   (yes)   (no)    (never) (100)
# ==========================================================================
smtp      inet  n       -       y       -       -       smtpd
#smtp      inet  n       -       y       -       1       postscreen
#smtpd     pass  -       -       y       -       -       smtpd
#dnsblog   unix  -       -       y       -       0       dnsblog
#tlsproxy  unix  -       -       y       -       0       tlsproxy
# Choose one: enable submission for loopback clients only, or for any client.
#127.0.0.1:submission inet n -   y       -       -       smtpd
submission inet n       -       y       -       -       smtpd
  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes
#  -o smtpd_tls_auth_only=yes
  -o smtpd_reject_unlisted_recipient=no
  -o smtpd_client_restrictions=permit_sasl_authenticated,reject
#  -o smtpd_helo_restrictions=$mua_helo_restrictions
#  -o smtpd_sender_restrictions=$mua_sender_restrictions
#  -o smtpd_recipient_restrictions=
#  -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
  -o milter_macro_daemon_name=ORIGINATING
# Choose one: enable smtps for loopback clients only, or for any client.
#127.0.0.1:smtps inet n  -       y       -       -       smtpd
smtps     inet  n       -       y       -       -       smtpd
  -o syslog_name=postfix/smtps
  -o smtpd_tls_wrappermode=yes
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_reject_unlisted_recipient=no
  -o smtpd_client_restrictions=permit_sasl_authenticated,reject
#  -o smtpd_helo_restrictions=$mua_helo_restrictions
#  -o smtpd_sender_restrictions=$mua_sender_restrictions
#  -o smtpd_recipient_restrictions=
#  -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
  -o milter_macro_daemon_name=ORIGINATING
#628       inet  n       -       y       -       -       qmqpd
pickup    unix  n       -       y       60      1       pickup
cleanup   unix  n       -       y       -       0       cleanup
qmgr      unix  n       -       n       300     1       qmgr
#qmgr     unix  n       -       n       300     1       oqmgr
tlsmgr    unix  -       -       y       1000?   1       tlsmgr
rewrite   unix  -       -       y       -       -       trivial-rewrite
bounce    unix  -       -       y       -       0       bounce
defer     unix  -       -       y       -       0       bounce
trace     unix  -       -       y       -       0       bounce
verify    unix  -       -       y       -       1       verify
flush     unix  n       -       y       1000?   0       flush
proxymap  unix  -       -       n       -       -       proxymap
proxywrite unix -       -       n       -       1       proxymap
smtp      unix  -       -       y       -       -       smtp
relay     unix  -       -       y       -       -       smtp
        -o syslog_name=postfix/$service_name
#       -o smtp_helo_timeout=5 -o smtp_connect_timeout=5
showq     unix  n       -       y       -       -       showq
error     unix  -       -       y       -       -       error
retry     unix  -       -       y       -       -       error
discard   unix  -       -       y       -       -       discard
local     unix  -       n       n       -       -       local
virtual   unix  -       n       n       -       -       virtual
lmtp      unix  -       -       y       -       -       lmtp
anvil     unix  -       -       y       -       1       anvil
scache    unix  -       -       y       -       1       scache
postlog   unix-dgram n  -       n       -       1       postlogd
#
# ====================================================================
# Interfaces to non-Postfix software. Be sure to examine the manual
# pages of the non-Postfix software to find out what options it wants.
#
# Many of the following services use the Postfix pipe(8) delivery
# agent.  See the pipe(8) man page for information about ${recipient}
# and other message envelope options.
# ====================================================================
#
# maildrop. See the Postfix MAILDROP_README file for details.
# Also specify in main.cf: maildrop_destination_recipient_limit=1
#
maildrop  unix  -       n       n       -       -       pipe
  flags=DRXhu user=vmail argv=/usr/bin/maildrop -d ${recipient}
#
# ====================================================================
#
# Recent Cyrus versions can use the existing "lmtp" master.cf entry.
#
# Specify in cyrus.conf:
#   lmtp    cmd="lmtpd -a" listen="localhost:lmtp" proto=tcp4
#
# Specify in main.cf one or more of the following:
#  mailbox_transport = lmtp:inet:localhost
#  virtual_transport = lmtp:inet:localhost
#
# ====================================================================
#
# Cyrus 2.1.5 (Amos Gouaux)
# Also specify in main.cf: cyrus_destination_recipient_limit=1
#
#cyrus     unix  -       n       n       -       -       pipe
#  flags=DRX user=cyrus argv=/cyrus/bin/deliver -e -r ${sender} -m ${extension} ${user}
#
# ====================================================================
# Old example of delivery via Cyrus.
#
#old-cyrus unix  -       n       n       -       -       pipe
#  flags=R user=cyrus argv=/cyrus/bin/deliver -e -m ${extension} ${user}
#
# ====================================================================
#
# See the Postfix UUCP_README file for configuration details.
#
uucp      unix  -       n       n       -       -       pipe
  flags=Fqhu user=uucp argv=uux -r -n -z -a$sender - $nexthop!rmail ($recipient)
#
# Other external delivery methods.
#
ifmail    unix  -       n       n       -       -       pipe
  flags=F user=ftn argv=/usr/lib/ifmail/ifmail -r $nexthop ($recipient)
bsmtp     unix  -       n       n       -       -       pipe
  flags=Fq. user=bsmtp argv=/usr/lib/bsmtp/bsmtp -t$nexthop -f$sender $recipient
scalemail-backend unix -       n       n       -       2       pipe
  flags=R user=scalemail argv=/usr/lib/scalemail/bin/scalemail-store ${nexthop} ${user} ${extension}
mailman   unix  -       n       n       -       -       pipe
  flags=FRX user=list argv=/usr/lib/mailman/bin/postfix-to-mailman.py ${nexthop} ${user}
```
:::

### /etc/opendkim.conf
:::spoiler
```shell=
victor-postfix@victorpostfix:~$ cat /etc/opendkim.conf
# This is a basic configuration for signing and verifying. It can easily be
# adapted to suit a basic installation. See opendkim.conf(5) and
# /usr/share/doc/opendkim/examples/opendkim.conf.sample for complete
# documentation of available configuration parameters.

Syslog			yes
SyslogSuccess		yes
#LogWhy			no

# Common signing and verification parameters. In Debian, the "From" header is
# oversigned, because it is often the identity key used by reputation systems
# and thus somewhat security sensitive.
Canonicalization	relaxed/simple
#Mode			sv
#SubDomains		no
OversignHeaders		From

# Signing domain, selector, and key (required). For example, perform signing
# for domain "example.com" with selector "2020" (2020._domainkey.example.com),
# using the private key stored in /etc/dkimkeys/example.private. More granular
# setup options can be found in /usr/share/doc/opendkim/README.opendkim.
#Domain			example.com
#Selector		2020
#KeyFile		/etc/dkimkeys/example.private

Domain	*

KeyFile /etc/opendkim/keys/default.private

Selector default

KeyTable /etc/opendkim/KeyTable


SigningTable /etc/opendkim/SigningTable
# In Debian, opendkim runs as user "opendkim". A umask of 007 is required when
# using a local socket with MTAs that access the socket as a non-privileged
# user (for example, Postfix). You may need to add user "postfix" to group
# "opendkim" in that case.
UserID			opendkim
UMask			007

# Socket for the MTA connection (required). If the MTA is inside a chroot jail,
# it must be ensured that the socket is accessible. In Debian, Postfix runs in
# a chroot in /var/spool/postfix, therefore a Unix socket would have to be
# configured as shown on the last line below.
#Socket			local:/run/opendkim/opendkim.sock
Socket			inet:8891@localhost
#Socket			inet:8891
#Socket			local:/var/spool/postfix/opendkim/opendkim.sock

#PidFile			/run/opendkim/opendkim.pid

# Hosts for which to sign rather than verify, default is 127.0.0.1. See the
# OPERATION section of opendkim(8) for more information.
InternalHosts		10.1.0.0/16, 10.0.0.0/16

# The trust anchor enables DNSSEC. In Debian, the trust anchor file is provided
# by the package dns-root-data.
TrustAnchorFile		/usr/share/dns/root.key
#Nameservers		127.0.0.1
```
:::

### /etc/opendkim/KeyTable
:::spoiler
```shell=
victor-postfix@victorpostfix:/etc/opendkim$ sudo cat KeyTable
[sudo] password for victor-postfix:
# OPENDKIM KEY TABLE
# To use this file, uncomment the #KeyTable option in /etc/opendkim.conf,
# then uncomment the following line and replace example.com with your domain
# name, then restart OpenDKIM. Additional keys may be added on separate lines.

#default._domainkey.example.com example.com:default:/etc/opendkim/keys/default.private
#selector1-tt3._domainkey.tt3.ecrm.com.tw tt3.ecrm.com.tw:selector1-tt3:/etc/opendkim/keys/tt3.ecrm.com.tw/selector1-tt3
#
selector1-tt5._domainkey.tt5.ecrm.com.tw tt5.ecrm.com.tw:selector1-tt5:/etc/opendkim/keys/tt5.ecrm.com.tw/selector1-tt5
```
:::

### /etc/opendkim/SigningTable
:::spoiler
```shell=
victor-postfix@victorpostfix:/etc/opendkim$ sudo cat SigningTable
# OPENDKIM SIGNING TABLE
# This table controls how to apply one or more signatures to outgoing messages based
# on the address found in the From: header field. In simple terms, this tells
# OpenDKIM "how" to apply your keys.

# To use this file, uncomment the SigningTable option in /etc/opendkim.conf,
# then uncomment one of the usage examples below and replace example.com with your
# domain name, then restart OpenDKIM.

# WILDCARD EXAMPLE
# Enables signing for any address on the listed domain(s), but will work only if
# "refile:/etc/opendkim/SigningTable" is included in /etc/opendkim.conf.
# Create additional lines for additional domains.

#*@example.com default._domainkey.example.com

# NON-WILDCARD EXAMPLE
# If "file:" (instead of "refile:") is specified in /etc/opendkim.conf, then
# wildcards will not work. Instead, full user@host is checked first, then simply host,
# then user@.domain (with all superdomains checked in sequence, so "foo.example.com"
# would first check "user@foo.example.com", then "user@.example.com", then "user@.com"),
# then .domain, then user@*, and finally *. See the opendkim.conf(5) man page under
# "SigningTable" for more details.

#example.com default._domainkey.example.com
# *@tt3.ecrm.com.tw selector1-tt3._domainkey.tt3.ecrm.com.tw

*@tt5.ecrm.com.tw selector1-tt5._domainkey.tt5.ecrm.com.tw
```
:::
![image](https://hackmd.io/_uploads/SkPcUTu5p.jpg)
