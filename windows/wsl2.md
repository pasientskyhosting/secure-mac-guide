## Install WSL

### Prerequisites
```
You must be running Windows 10 version 2004 and higher (Build 19041 and higher) or Windows 11.
```

### Install
You can now install everything you need to run Windows Subsystem for Linux (WSL) by entering this command in an administrator PowerShell or Windows Command Prompt and then restarting your machine.
```shell
wsl --install
```

It will automatically install all needed features, and make first wsl install with default ubuntu

### Create user
Once the process of installing your Linux distribution with WSL is complete, open the distribution (Ubuntu by default) using the Start menu. You will be asked to create a User Name and Password for your Linux distribution.
That will be the default user when you start ubuntu
After your user has been made, and you are logged in for first time, remember to update
```shell
sudo apt update
sudp apt upgrade
```

### Login as root if you fucked up sudo or other reasons
```shell
wsl -u root
```

### Setup terminal
Install [Terminal](https://github.com/microsoft/terminal) from Microsoft for better interface with you wsl installations.  




