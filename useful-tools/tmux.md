# tmux

### Tmux

#### Description

Tmux is a tool set which creates a local session on the server that you are on. This means that you can:

* disconnect and reconnect to your session
* leave something running in the session and not care if your session is interrupted by a network interruption or power cut at your end
* share your session with anyone else logged in on the server ( as the same user ). This can be useful for debugging, shared work or educational pieces

#### Installation

```bash
yum install tmux -y
```

#### usage commands

*   create a new tmux session

    ```bash
    tmux
    ```
*   create a new tmux session but give it a name

    ```bash
    tmux new -s < name of session >
    ```
*   re-attach to the last tmux session in use

    ```bash
    tmux attach
    ```
*   list tmux sessions available to you

    ```bash
    tmux ls
    ```
*   attach to a specific tmux session

    ```bash
    tmux attach -t < name of session >
    ```

#### Notes and other

Tmux sessions are local to the user - if you want to share your session with someone they have to be logged in as the same user

tmux sessions aren’t reboot consistent by default. There are ways however

If regularly creating the same tmux sessions, say for a cluster then I wrote a script

[https://github.com/tuckertt/random\_scripts/tree/master/tmux\_session](https://github.com/tuckertt/random_scripts/tree/master/tmux_session)

there are plugins….

[https://github.com/tmux-plugins/tpm](https://github.com/tmux-plugins/tpm)

#### Commands

| \<ctrl + b>                                | This is your primary method of controlling the session, telling tmux you want to type to it rather than into the session you’re using. This is the default prefix and can be changed                      |
| ------------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| \<ctrl + b> ;; d                           | **detach from the current session leaving it running in the background**                                                                                                                                  |
| \<ctrl + b> ;; x                           | **kill pane / session**                                                                                                                                                                                   |
| \<ctrl + b> ;; %                           | **split the screen vertically, can be used as many times as you like to sub divide the current pane**                                                                                                     |
| \<ctrl + b> ;; “                           | **split the screen horizontally, can be used as many times as you like to sub divide the current pane**                                                                                                   |
| \<ctrl + b> ;; “ “                         | **hitting space bar will allow tmux to re-arrange the panes, hit it enough times and it will lay the panes out so they’re spread out equally**                                                            |
| \<ctrl + b> ;; < arrow key >               | **allows you to move around the various panes left towards the left, right goes right etc**                                                                                                               |
| <**ctrl** + b> ;; < arrow key repeatedly > | if you hold the ctrl key after hitting “b” then repeatedly hit an arrow key it will resize the current panes size in the direction of the arrow                                                           |
| \<ctrl + b> ;; :setw synchronize-panes on  | **(note the colon ) sets the window you’re currently in to have the panes data entry synchronised - type once but enter the same command to 50 machines. Can lead to pain if wanting to reboot a server** |
| \<ctrl + b> ;; :setw synchronize-panes on  | **(note the colon ) sets pane synchronisation off**                                                                                                                                                       |
| \<ctrl + b> ;; c                           | creates a new window ( windows with numbers can be seen at the bottom                                                                                                                                     |
| \<ctrl + b> ;; n                           | moves to the next window if you have more than one available                                                                                                                                              |
| \<ctrl + b> ;; p                           | moves to the previous window if more tahn one are available                                                                                                                                               |
| \<ctrl + b> ;; \<number>                   | moves to the window number provided ( windows can be seen on the bottom bar )                                                                                                                             |
| \<ctrl + b> ;; ,                           | give your current window a name / change the current one                                                                                                                                                  |
| \<ctrl + b> ;; {                           | **allows you to scroll in the pane (q to quit scrolling)**                                                                                                                                                |
| \<ctrl + b> ;; :set -g mouse on            | use shift for mouse based fun                                                                                                                                                                             |
| \<ctrl + b> ;; :set -g mouse off           | go back to your text adventure                                                                                                                                                                            |
| \<ctrl + b> ;; z                           | **zoom in or out of the focused pane. Switches off sync’d panes**                                                                                                                                         |
|                                            |                                                                                                                                                                                                           |
