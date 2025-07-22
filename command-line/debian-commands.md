# Debian Commands

## General commands

text that appears within angled braces “ <> ” is a command or a suggested non text button to hit i.e

* “ < your name here > “ ⇒ put your name between the double quotes
* < tab > ⇒ hit the tab button

| Area of interest | command            | usage / sub commands / commands inside app | what it does                                                                          | how is it useful                                                                                                                                                     |
| ---------------- | ------------------ | ------------------------------------------ | ------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Disk usage       | df -h              |                                            | shows disk usage in a human readable format                                           | lets you identify if a disks usage is increasing / decreasing / what percentage is used                                                                              |
|                  | du -h —max-depth=1 |                                            | from a given directory gives you an understanding of the sub directories beneath it   | very useful if attempting to find out where disk space is being consumed - go to a directory and run the command for each large directory down to find where data is |
|                  |                    |                                            |                                                                                       |                                                                                                                                                                      |
| text editing     | vim                |                                            | a powerful text editor, similar to vi but with colours and context awareness          | edit files                                                                                                                                                           |
|                  |                    | :                                          | command operator - anything that comes after if vim expects is a command for it to do |                                                                                                                                                                      |
|                  |                    | :colo \<tab>                               | allows you to change the colour scheme of the text in vim                             | useful if the default for the session is hard to read                                                                                                                |
|                  |                    | :colo < colour scheme name >               | allows you to specify the name of the colour scheme rather than scrolling through     |                                                                                                                                                                      |
|                  |                    | :w                                         | writes an update to the file                                                          |                                                                                                                                                                      |
|                  |                    | :q                                         | quits the file                                                                        |                                                                                                                                                                      |
|                  |                    | :q!                                        | force quits the file ignoring save prompt                                             |                                                                                                                                                                      |
|                  |                    | :wq                                        | writes the file changes to disk then quits                                            |                                                                                                                                                                      |
|                  |                    | :w !sudo tee %                             | writes the contents of the file to the file using the sudo command                    | If you’ve edited a file as the wrong user then allows you to save it as the root user without losing your work                                                       |
|                  |                    |                                            |                                                                                       |                                                                                                                                                                      |
| file location    | which              | which df                                   | lets you know where the program you’re calling lives / what hte full path is          | If you’re trying to figure out where a package has been deployed and it’s not in the defaults it can be very                                                         |
|                  | $PATH              | echo $PATH                                 | shows where the operating will look by default when you try running a command         |                                                                                                                                                                      |
|                  |                    |                                            |                                                                                       |                                                                                                                                                                      |
|                  |                    |                                            |                                                                                       |                                                                                                                                                                      |
|                  |                    |                                            |                                                                                       |                                                                                                                                                                      |
|                  |                    |                                            |                                                                                       |                                                                                                                                                                      |
|                  |                    |                                            |                                                                                       |                                                                                                                                                                      |
|                  |                    |                                            |                                                                                       |                                                                                                                                                                      |
|                  |                    |                                            |                                                                                       |                                                                                                                                                                      |
|                  |                    |                                            |                                                                                       |                                                                                                                                                                      |
|                  |                    |                                            |                                                                                       |                                                                                                                                                                      |
|                  |                    |                                            |                                                                                       |                                                                                                                                                                      |

## Update Operating system version

### Debian

Debian based distros often have a shorter life cycle than other systems ( like RHEL / Rocky ) so some packages may not be available for the OS unless you upgrade the distribution in use

IF NERVOUS CREATE A DISK SNAPSHOT TO RETURN TO ( REMEMBER TO DELETE IT ON COMPLETION )

#### Buster ( Debian 10 )→ Bullseye ( Debian 11 )

1.  make sure the operating system is up to date

    ```bash
    sudo apt update; sudo apt upgrade
    ```
2.  Clean up any extra dependencies that are no longer required

    ```bash
    sudo apt autoremove
    ```
3.  replace references to Buster in the apt source list with Bullseye

    ```bash
    sudo sed -i 's/buster/bullseye/g' /etc/apt/sources.list
    ```
4.  amend the security updates line so that it follows the standard

    ```bash
    sudo vim /etc/apt/sources.list
    ```

    1. On the security line change “ **/updates** “ to “ **-security** “
5.  get apt to update to make itself aware of the changes

    ```bash
    **sudo apt update** 
    ```
6.  update teh current setup without the added complication of new packages

    ```bash
    **sudo apt upgrade --without-new-pkgs -y** 
    ```

    * for more control select no for the package restart dialogue box that appears
      * cron - yes fine to restart
7.  run the full upgrade now

    ```bash
    **sudo apt full-upgrade -y**
    ```

    * ssh\_cron - fine to restart, connection shouldn’t drop
    * for each package version keep the local version so working configurations aren’t overridden
8.  restart the system

    ```bash
    sudo init 6 
    or 
    sudo reboot 
    or 
    sudo shutdown —now 

    ( however you prefer )
    ```
9.  Once the system comes back up reconnect and check that the system is up to date

    ```bash
    sudo apt update; sudo apt upgrade

    ```

    1. with luck nothing comes back to be updated
10. remove the now unused packages

    ```bash
    sudo apt autoremove -y
    ```

your system should now be up to date and on Debian 11 - Bullseye

#### Bullseye ( Debian 11 )→ Bookworm ( Debian 12 )

1.  make sure the operating system is up to date

    ```bash
    sudo apt update; sudo apt upgrade
    ```
2.  Clean up any extra dependencies that are no longer required

    ```bash
    sudo apt autoremove
    ```
3.  replace references to Buster in the apt source list with Bullseye

    ```bash
    sudo sed -i 's/bullseye/bookworm/g' /etc/apt/sources.list
    ```
4.  get apt to update to make itself aware of the changes

    ```bash
    **sudo apt update** 
    ```
5.  update teh current setup without the added complication of new packages

    ```bash
    **sudo apt upgrade --without-new-pkgs -y** 
    ```

    * for more control select no for the package restart dialogue box that appears
      * ssh\_cron - yes fine to restart
6.  run the full upgrade now

    ```bash
    **sudo apt full-upgrade -y**
    ```

    * ssh\_cron - fine to restart, connection shouldn’t drop
    * for each package version keep the local version so working configurations aren’t overridden
7.  restart the system

    ```bash
    sudo init 6 
    or 
    sudo reboot 
    or 
    sudo shutdown —now 

    ( however you prefer )
    ```
8.  Once the system comes back up reconnect and check that the system is up to date

    ```bash
    sudo apt update; sudo apt upgrade

    ```

    1. with luck nothing comes back to be updated
9.  remove the now unused packages

    ```bash
    sudo apt autoremove -y
    ```

your system should now be up to date and on Debian 12 - Bookworm
