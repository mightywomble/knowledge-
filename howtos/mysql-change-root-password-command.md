# MySQL Change root Password Command

## [https://www.cyberciti.biz/faq/mysql-change-root-password/](https://www.cyberciti.biz/faq/mysql-change-root-password/)

Author: Vivek Gite Last updated: December 9, 2024 [142 comments](https://www.cyberciti.biz/faq/mysql-change-root-password/#comments)[![See all MySQL Database Server related FAQ](https://www.cyberciti.biz/media/new/category/old/mysqllogo.gif)](https://www.cyberciti.biz/faq/category/mysql/)How do I change MySQL root password under Linux, FreeBSD, OpenBSD and UNIX-like like operating system over the ssh session?\
\
[![See all UNIX related articles/faq](https://www.cyberciti.biz/media/new/category/old/unix-logo.gif)](https://www.cyberciti.biz/faq/category/unix/)Setting up MySQL password is one of the essential tasks. By default, root user is MySQL admin account user. Please note that the Linux or UNIX root account for your operating system and MySQL root user accounts are different. They are separate, and nothing to do with each other. Sometime you may remove Mysql root account and setup admin user as super user for security purpose.\


| Tutorial details  |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| ----------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Difficulty level  | [Easy](https://www.cyberciti.biz/faq/tag/easy/)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| Root privileges   | No                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| Requirements      | Linux or Unix terminal                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| Category          | [Database Server\|](https://www.cyberciti.biz/faq/mysql-change-root-password/#Database_Server\|)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| Prerequisites     | mysqladmin or mysql command                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| OS compatibility  | [AIX](https://www.cyberciti.biz/faq/mysql-change-root-password/) • AlmaLinux • [Alpine](https://www.cyberciti.biz/faq/category/alpine-linux/) • [Amazon Linux](https://www.cyberciti.biz/faq/category/aws-ec2-ebs-cloudfront/) • [Arch](https://www.cyberciti.biz/faq/category/arch-linux/) • BSD • [CentOS](https://www.cyberciti.biz/faq/category/centos/) • [Debian](https://www.cyberciti.biz/faq/category/debian-ubuntu/) • [Fedora](https://www.cyberciti.biz/faq/category/fedora-linux/) • [FreeBSD](https://www.cyberciti.biz/faq/category/freebsd/) • [HP-UX](https://www.cyberciti.biz/faq/category/hp-ux-unix/) • [Linux](https://www.cyberciti.biz/faq/category/linux/) • [macOS](https://www.cyberciti.biz/faq/category/mac-os-x/) • Mint • Mint • NetBSD • [OpenBSD](https://www.cyberciti.biz/faq/category/openbsd/) • [openSUSE](https://www.cyberciti.biz/faq/tag/opensuse/) • Pop!\_OS • [RHEL](https://www.cyberciti.biz/faq/category/redhat-and-friends/) • Rocky • [Slackware](https://www.cyberciti.biz/faq/tag/slackware-linux/) • [Stream](https://www.cyberciti.biz/faq/tag/centos-stream/) • [SUSE](https://www.cyberciti.biz/faq/category/suse/) • [Ubuntu](https://www.cyberciti.biz/faq/category/ubuntu-linux/) • [Unix](https://www.cyberciti.biz/faq/category/unix/) • WSL |
| Est. reading time | 5 minutes                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |

### Method #1: Using mysqladmin command to change root password for MySQL server

**WARNING!** It would help if you were always careful when changing existing root or any other mysql user account password. You must update existing apps and shell/perl/python/php scripts files to match the new password. Any wrong settings will result in downtime. The author or nixCraft is not responsible for any downtime or wrong settings.If you have never set a root password for MySQL server, the server does not require a password at all for connecting as root. To setup root password for first time, use mysqladmin command at shell prompt as follows:\
`$ mysqladmin -u root password NEWPASSWORD`\
However, if you want to change (or update) a root password, then you need to use the following command:\
`$ mysqladmin -u root -p'oldpassword' password newpass`\
You can state hostname or IP address too. The syntax is:\
`$ mysqladmin -u root -h {IP_or_Host} -p'{oldpassword}' password {NewPassWord}`

#### Examples

For example, If the old password is abc, you can set the new password to 123456, enter:

```
$ mysqladmin -u root -p'abc' password '123456'
```

OR state 127.0.0.1 as MySQL server IP :

```
$ mysqladmin -u root -h 127.0.0.1 -p'abc' password '123456'
```

Note:123456 password is used for demonstration purpose only. You must select a strong password. It is an important protection to help you have safer MySQL database transactions.

#### How to securely change the password for the MySQL server

In this example, mysqladmin will first prompt for the old password, and then you can set a new password. This way, your password can not be listed in the ps or top or any other Linux and Unix command or saved in the shell history:\
`$ mysqladmin -u root -p password`\
Sample outputs:

```
Enter password: 
New password: 
Confirm new password: 
```

You may see warning as follows:\


Warning: Since password will be sent to server in plain text, use ssl connection to ensure password safety.

To avoid that, try the following option when connecting remote MySQL server or RDS via an IP address or hostname\
`$ mysqladmin -h {server_ip_here} --ssl-mode=REQUIRED -u root -p password`\
OR\
`$ mysqladmin --ssl-mode=VERIFY_CA -h {server_ip_here} -u root -p password`\
For example:\
`$ mysqladmin --ssl-mode=VERIFY_CA -h 10.8.0.1 -u root -p password`\
The --ssl-mode=REQUIRED option tells the mysql/mysqladmin command establish an encrypted connection if the server supports encrypted connections. The connection attempt fails if an encrypted connection cannot be established. On other hand, the --sl-mode=VERIFY\_CA option is just like REQUIRED option, but additionally verify the server Certificate Authority (CA) certificate against the configured CA certificates. The connection attempt fails if no valid matching CA certificates are found.

**Sample live session from my home server using mysqladmin**

[![Fig.01: mysqladmin command in action](https://www.cyberciti.biz/media/new/images/faq/2013/05/mysqladmin-change-root-password-demo.png)](https://www.cyberciti.biz/media/new/images/faq/2013/05/mysqladmin-change-root-password-demo.png)

Fig.01: mysqladmin command in action

#### How do I verify that the new password is working or not?

Use the following mysql command:\
`$ mysql -u root -p'123456' db-name-here`\
OR\
`$ mysql -u root -p'123456' -e 'show databases;'`

#### A note about changing MySQL password for other users

To change a normal user password you need to type the following command. In this example, change the password for ‘nixcraft’ mysql user:\
`$ mysqladmin -u nixcraft -p'nixcrafts-old-password' password new-nixcraft-password`

### Method #2: Changing MySQL root user password using the mysql command

This is an another method. MySQL stores username and passwords in user table inside MySQL database. You can directly update or change the password using the following method for user called nixcraft:

#### Login to mysql server, type the following command at shell prompt:

`$ mysql -u root -p`

#### Use mysql database (type command at your mysql> prompt):

```
mysql>  use mysql;
```

To view the current permissions for the ‘nixcraft’ mysql user, run:

```
mysql>  SHOW GRANTS FOR 'nixcraft'@'localhost';
```

#### Change password for user nixcraft, enter:

The syntax for an older MySQL server:

```
mysql>  update user set password=PASSWORD("NEWPASSWORD") where User='nixcraft';
```

The syntax for the latest version of the MySQL server 8.x:

```
mysql>  ALTER USER 'nixcraft'@'%' IDENTIFIED BY 'NEWPASSWORD';
```

#### Finally, reload the privileges:

```
mysql>  flush privileges;
mysql>  quit
```

**Sample live session from my home server**

[![Fig.02: Changing mysql password for nixcraft user using sql commands.](https://www.cyberciti.biz/media/new/images/faq/2013/05/mysql-sql-command-change-password-demo.png)](https://www.cyberciti.biz/media/new/images/faq/2013/05/mysql-sql-command-change-password-demo.png)

Fig.02: Changing mysql password for nixcraft user using sql commands.

This method is also useful with PHP, Python, or Perl scripting APIs.

**Sample live session from my home server to change the mysql root account password**

Login to the server using TCP/IP with SSL mode and existing password:\
`$ mysql --ssl-mode=REQUIRED -u root -h 10.8.0.1 -p`\
Then:

```
mysql> USE mysql;
# LIST permissions/grants for the 'root' user
mysql> SHOW GRANTS FOR 'root'@'%';
mysql> SHOW GRANTS FOR 'root'@'localhost';
mysql> SHOW GRANTS FOR 'root'@'10.8.0.1';
# NOW CHANGE IT as per needs
mysql> ALTER USER 'root'@'%' IDENTIFIED BY 'NEW-PASSWORD-HERE';
mysql> FLUSH PRIVILEGES; 
mysql> QUIT;
```

Verify it:\
`$ mysql --ssl-mode=REQUIRED -u root -h 10.8.0.1 -p`\
When promoted type the new password such as ‘NEW-PASSWORD-HERE’.

### Summing up

And that is how you change your MySQL server root (admin) account password. I strongly suggest that you make or update ypur .my.cnf file as follows (this is optional but recommend for all DBAs or Linux/Unix sysadmins). I’m assuming that Unix / Linux root account is also responsible for managing the MySQL server:

1. Create SSL CA bundle file:\
   `$ sudo` [`cat`](https://www.cyberciti.biz/faq/linux-unix-appleosx-bsd-cat-command-examples/) `/var/lib/mysql/ca.pem /var/lib/mysql/client-cert.pem >/root/mysql-combined-ca-bundle.pem`
2. Set permissions using the chmod command and chown command:\
   `$ sudo chmod -v 0600 /root/mysql-combined-ca-bundle.pem`\
   `$ chown -v root:root /root/mysql-combined-ca-bundle.pem`
3.  Finally create \~/.my.cnf file and set permissions replacing the password and hostname/IP address as per your needs:

    ```
    [client]
    user=root
    password='Your-PASSWORD-HERE'
    host=SERVER_IP_HERE
    port=3306
    ssl-mode=VERIFY_CA 
    ssl-ca=/root/mysql-combined-ca-bundle.pem
    ```

    And permission:\
    `$ sudo chmod -v 0600 /root/.my.cnf`\
    `$ sudo chown -v root:root /root/.my.cnf`
4. Here is how to connect it:\
   `$ sudo mysql`

#### A note about the mysql\_config\_editor command

You can create .mylogin.cnf file using the mysql\_config\_editor command as follows which is a little secure as it hides password on screen:\
`$ mysql_config_editor set \`\
`--login-path=remote \`\
`--host={SERVER_IP} \`\
`--port={SERVER_PORT} \`\
`--user=root \`\
`--password`

#### See also:

* [HowTo: Recover MySQL root account password](https://www.cyberciti.biz/tips/recover-mysql-root-password.html)
* See locally installed man pages using the [man command](https://bash.cyberciti.biz/guide/Man_command)/[help command](https://bash.cyberciti.biz/guide/Help_command) for mysqladmin(1) and mysql(1) commands for more info:\
  `$ man 1 mysql`\
  `$ man 1 mysqladmin`\
  `$ man 1 mysql_config_editor`

🥺
