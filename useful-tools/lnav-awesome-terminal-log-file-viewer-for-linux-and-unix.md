# lnav – Awesome terminal log file viewer for Linux and Unix

It is no secret that whether you are a developer or sysadmin, you need to use log files to troubleshoot errors on your Linux and Unix systems. You use tools like grep, tail, cat, or journalctl to view log files. However, you may need help with so many log files. These essential Unix tools are suitable for basic text but fall short when dealing with many log files. You can get tired from sifting through endless lines of log files. The lnav utility is here to the rescue! It is a powerful log file viewer that goes beyond the basics. It understands your logs by identifying timestamps, log levels, and other vital details. You can run SQLite SQL queries against your standard log files and build reports for your needs. Let us see how to install and use the lnav tool quickly.\


[![lnav turning Nginx logs into SQLite SQL outputs](https://www.cyberciti.biz/media/new/cms/2024/06/lnav-welcome-599x185.png)](https://www.cyberciti.biz/media/new/cms/2024/06/lnav-welcome.png)

lnav turning Nginx logs into SQLite SQL outputs

### What unique features does the lnav utility offer?

1. You can decompress your log files as needed on the fly, similar to the z\* utilities on Linux and Unix.
2. Detect log file format.
3. Merge the files together by time into a single view.
4. Terminal color support makes it easier to spot errors and warnings.
5. SSH (SFTP) support for remote Linux and Unix machines to view log files.
6. Tail the files follow renames, find new files in directories when log files are rotated when you are doing troubleshooting.
7. It can build an index of errors and warnings.
8. Show pretty-print JSON-lines if needed.
9. You can jump quickly to the previous/next error.
10. Search using regular expressions.
11. Highlight text with a regular expression.
12. Filter messages using regular expressions or SQLite expressions.
13. Pretty-print structured text.
14. View a histogram of messages over time.
15. Query messages using SQLite.
16. In short, one utility covers functions of core utilities like grep, cat, tail, and more.

### Installation

Type installation commands as per your Linux and Unix variants.

#### Debian/Ubuntu Linux

Try the “[apt,](https://www.cyberciti.biz/faq/ubuntu-lts-debian-linux-apt-command-examples/)” or “[apt-get](https://www.cyberciti.biz/tips/linux-debian-package-management-cheat-sheet.html)” command as follows:\
`$ sudo apt install lnav`

#### CentOS/RHEL/Fedora/Rocky/Alma/Oracle Linux

[First, enable the EPEL repo](https://www.cyberciti.biz/faq/install-epel-repo-on-an-rhel-8-x/) (except for Fedora Linux), then use the following dnf command to install lnav:\
`$ sudo dnf install lnav`

#### Arch Linux

Use the pacman command:\
`$ sudo pacman -S lnav`

#### Alpine Linux

Use the following [apk command](https://www.cyberciti.biz/faq/10-alpine-linux-apk-command-examples/):\
`# apk add lnav`

#### OpenSUSE / SUSE Linux

You can try the following zypper command:\
`$ sudo zypper install lnav`

#### macOS installing lnav

First, [enable and install homebrew](https://www.cyberciti.biz/faq/how-to-install-homebrew-on-macos-package-manager/), then type brew command:\
`$ brew install lnav`\
OR try the port command:\
`$ sudo port install lnav`

#### FreeBSD Unix install lnav

Try the pkg command as follows:\
`$ pkg install lnav`

### How do I use the lnav terminal-based log file navigator utility?

The syntax is simple:\
`# Log file`\
`$ lnav /path/to/file.log`\
`$ lnav /path/to/file1.log /path/to/file2.log`\
`# Dir name`\
`$ lnav /path/to/your/app/log/dir1/`\
`$ lnav /path/to/your/app/log/dir1/ /var/log/`\
`# Wildcard`\
`$ lnav /var/log/nginx/app_*_error*log`\
`$ lnav /var/log/nginx/app_*_error*log /var/log/*.err`\
`##############################`\
`# Using ssh for remote hosts #`\
`##############################`\
`$ lnav user@server-name-here:/var/log/file.log`\
`$ lnav vivek@server1.cyberciti.biz:/var/log/`\
`$ lnav vivek@server1.cyberciti.biz:/var/log/*.err`\
`#########################################`\
`# Linux system with systemd-journald use`\
`# lnav as the pager`\
`#########################################`\
`$ journalctl | lnav`\
`$ journalctl -f | lnav`\
`$ journalctl -u ssh.service | lnav`\
\


[![lnav – Awesome terminal log file viewer for Linux and Unix showing ssh.service log on Ubuntu](https://www.cyberciti.biz/media/new/cms/2024/06/lnav-%E2%80%93-Awesome-terminal-log-file-viewer-for-Linux-and-Unix-showing-ssh.service-log-on-Ubuntu-599x384.jpg)](https://www.cyberciti.biz/media/new/cms/2024/06/lnav-%E2%80%93-Awesome-terminal-log-file-viewer-for-Linux-and-Unix-showing-ssh.service-log-on-Ubuntu.jpg)

Click to enlarge

To find errors, press <kbd>e</kbd> and it will move to the next error. You can hit the <kbd>Shift+E</kbd> to move to the previous error. Similarly <kbd>w</kbd> and <kbd>Shift+W</kbd> are used for move to next or previous warning respectively. Press <kbd>q</kbd> or <kbd>CTRL+c</kbd> to exit from the lnav session or command prompt. When opening SFTP URLs, if the password is not provided for the remote host, the [SSH agent can be used to do authentication](https://www.cyberciti.biz/faq/how-to-use-ssh-agent-for-authentication-on-linux-unix/). To search for text in files, you can press <kbd>/</kbd> to enter the search prompt. You can hit the <kbd>TAB</kbd> to autocomplete search string.

#### Viewing Docker container log on Linux

The simplest syntax is as follows:\
`$ docker logs container-id | lnav`\
`$ docker logs -f container-id | lnav`\
If container ID is 611ac85cc97d or it is called “app”, then:\
`$ docker logs 611ac85cc97d | lnav`\
`$ docker logs -f app | lnav`\
The latest version of lnav supports the following docker:// URL syntax too:\
`$ lnav docker://{container_id_or_name}/path/to/log/file`\
`$ lnav docker://{container_id_or_name}/var/dir1`\
`$ lnav docker://app/var/log/`\
`$ lnav docker://app/var/log/nginx/nginx.app.log`\


[![docker logs viewed with lnav](https://www.cyberciti.biz/media/new/cms/2024/06/docker-logs-viewed-with-lnav-599x385.png)](https://www.cyberciti.biz/media/new/cms/2024/06/docker-logs-viewed-with-lnav.png)

Click to enlarge

#### Watching the output of any command

Many commands generate outputs and logs while working. For example, here is how to watch the output of make command when compiling something:\
`$ lnav -e 'make -j8'`\
The <kbd>-e command1</kbd> is used to execute a shell command named command1.

#### Using SQLite interface

In lnav, log analysis can be performed using the SQLite interface. This is a killer feature. All log messages are accessible through virtual tables that are generated for each file format. These tables are named after the log format, and each log message is represented as a separate row. For instance, consider the following log message from an Nginx access log:\
`$ lnav /var/log/nginx/www.cyberciti.biz_https_access.log`\
Now you will see\


[![Nginx log with lnav](https://www.cyberciti.biz/media/new/cms/2024/06/nginx_log_1-599x44.png)](https://www.cyberciti.biz/media/new/cms/2024/06/nginx_log_1.png)

Click to enlarge

You can activate the SQL prompt by pressing the <kbd>;</kbd> key. Now you can write a simple query. For example:

```
SELECT * FROM logline LIMIT 10
```

And you will get results:\


[![Nginx log file sql output](https://www.cyberciti.biz/media/new/cms/2024/06/nginx_log_2-599x59.png)](https://www.cyberciti.biz/media/new/cms/2024/06/nginx_log_2.png)

Click to enlarge

Don’t worry. There will be a help window to explain SQL fields and examples, too. As long as you know SQL statements, you can build queries. You can extract and save this data. Numerous options, including commands and advanced SQL syntax, are available to meet your needs, so bookmarking the [documentation URL is a](https://docs.lnav.org/) good idea.

### Summing up

The lnav is not just simple log file viewers. It offers advanced features that one can use to run SQL queries using SQLite and build reports with log files for various reasons, such as troubleshooting or getting information about app usage from log files. The TUI is easy to use. Apart from SQL support, you can search using regex, colorful highlighting for essential data, support for various popular log files, Linux containers, and remote file viewing using SSH. I found this tool handy and highly recommend it to Linux/Unix sysadmins and developers. You can try out lnav as follows or see the [project home page here](https://lnav.org/):\
`# Start Basic Tutorial:`\
`$ ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password tutorial1@demo.lnav.org`\
`# Playground:`\
`$ ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password playground@demo.lnav.org`

🥺 Was this helpful? Please add [a comment to show your appreciatio](https://www.cyberciti.biz/open-source/lnav-linux-unix-ncurses-terminal-log-file-viewer/#respond)
