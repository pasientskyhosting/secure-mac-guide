## GPG - Yubikey integration

---
### Initial guide

#### Guides for wsl2 yubikey and gpg
[Guide part 1](https://jardazivny.medium.com/the-ultimate-guide-to-yubikey-on-wsl2-part-1-dce2ff8d7e45)
It will focus on setting up yubikey and gpg on windows  
[Guide part 2](https://jardazivny.medium.com/the-ultimate-guide-to-yubikey-on-wsl2-part-2-1d9546ef23a6)
Focusing on getting gpg and ssh key from windows and yubikey into your wsl distribution
[Guide part 3](https://jardazivny.medium.com/the-ultimate-guide-to-yubikey-on-wsl2-part-3-974a08b4853b)
Focusing on your git ssh key and encryption key. Both comes from your yubikey

---
### Gopass 
>  Ensure your git checkout works first.  

First ensure you have go language installed in your wsl in stance
```shell
  sudo -i
  rm -rf /usr/local/go && tar -C /usr/local -xzf go1.18.linux-amd64.tar.gz
```

Add the path to your environment, if not already there. 
```shell
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
. ~/.bashrc
```

Install gopass
```shell
go install github.com/gopasspw/gopass
gopass setup
gopass clone git@github.com:pasientskyhosting/pass-ansible.git infrastructure
gopass clone git@github.com:pasientskyhosting/pass-ansible-production.git infrastructure-production
```

If you want gopass to be able to copy to clipboard in windows from wsl
```shell
echo 'export GOPASS_CLIPBOARD_COPY_CMD="xclip"' >> ~/.bashrc
echo 'GOPASS_CLIPBOARD_CLEAR_CMD="xclip"' >>  ~/.bashrc
```

And create fil in your path with name xclip and make sure its executable
```shell
#!/bin/bash
cat | clip.exe
```


And test it 
```shell
gopass generate --clip test
```

[Internal doc](https://github.com/pasientskyhosting/ansible-infrastructure#password-management-with-gopass)