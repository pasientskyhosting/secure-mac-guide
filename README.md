This is a practical guide to using [YubiKey](https://www.yubico.com/faq/yubikey/) as a SmartCard for storing GPG encryption and signing keys.
An authentication key can also be created for SSH and used with gpg-agent.
Keys stored on a smartcard like YubiKey seem more difficult to steal than ones stored on disk, and are convenient for everyday use.

The blog "[Exploring Hard Tokens](https://www.avisi.nl/blog/2012/01/05/exploring-hard-tokens/)" describes the disadvantages of the combination of a username/password for acces control. Passwords can be cracked or retrieved by social engineering. They can be read from faulty systems or even retrieved from unsecured internet access.

Authentication on a workstation often is done by using a username and password. Furthermore, it is almost impossible to detect when an attacker accesses a system. Therefore it is important to strengthen your authentication by adding a second step to your authentication process.

<!-- TOC depthFrom:1 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [Purchase YubiKey](#purchase-yubikey)
- [Prepare your Macbook](#prepare-your-macbook)
- [Install required software](#install-required-software)
	- [Install Homebrew](#install-homebrew)
	- [Install GPG](#install-gpg)
	- [Install Yubikey Personalization Tools](#install-yubikey-personalization-tools)
	- [Install PAM Yubico](#install-pam-yubico)
- [Enable PAM on your Macbook](#enable-pam-on-your-macbook)
	- [Disable System Integrity Protection](#disable-system-integrity-protection)
	- [Copy the PAM Module](#copy-the-pam-module)
	- [Enable System Integrity Protection](#enable-system-integrity-protection)
- [Configure your Yubikey](#configure-your-yubikey)
- [Change Yubikey Mode](#change-yubikey-mode)
- [Configure PAM on your Macbook](#configure-pam-on-your-macbook)
- [Prepare GPG](#prepare-gpg)
- [Enable Yubikey for Auth, Sudo and Screensaver](#enable-yubikey-for-auth-sudo-and-screensaver)
- [Generating More Secure GPG Keys](#generating-more-secure-gpg-keys)
	- [Generating the Primary Key](#generating-the-primary-key)
- [References](#references)

<!-- /TOC -->

# Purchase YubiKey
We use the Yubikey Neo as it features as a Smartcard and has options for NFC.

You should also buy a regular Yubikey as a backup key for your computer login, because if you lose your Neo, you wont be able to login into your computer.

https://www.dustin.dk/product/5010851791/yubikey-neo
https://www.dustin.dk/product/5010894443/yubikey-4

Consider purchasing a pair and programming both in case of loss or damage to one of them.

# Prepare your Macbook
Please make sure before you start this proces, that your Macbook has enabled FileVault 2 disk encryption.
Apple has an exelent guide here https://support.apple.com/en-gb/HT204837

When you have verified your computer has disk encryption enabled, and you use a strong password, start preparing your laptop for the Yubikey. This guide will allow you to enable the Yubikey as 2-Factor authentication for you laptop - this means you need to have the Yubikey plugged in, to login to your computer, while it will also help you configure your yubikey as a smartcard where you will store your SSH key you use to acces services.

# Install required software
The required software for this guide is:
* Homebrew
* PAM Yubico
* Yubikey personalization tools
* GPG

## Install Homebrew
Open a Terminal window and then run the following command to install Homebrew:

```
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

## Install GPG
Open a Terminal window and then run the following commands:
```
brew tap homebrew/versions
brew install gnupg21 pinentry-mac ykneomgr
```

## Install Yubikey Personalization Tools
Install the latest version of the YubiKey Personalization Tool from the App Store -
https://itunes.apple.com/us/app/yubikey-personalization-tool/id638161122?mt=12

## Install PAM Yubico
Open a Terminal window, and run the following command:
```
brew install yubico-pam
```

# Enable PAM on your Macbook
Mac OS X 10.11 (El Capitan) introduced a new security feature, System Integrity Protection
(AKA “rootless”). The feature protects certain directories from being modified. In order for the
OS X login to function in version 10.11, a file required for the Yubico PAM module to function
(pam_yubico.so) needs to be moved to a directory protected by System Integrity Protection.
To resolve this issue, it is necessary to temporarily disable System Integrity Protection, move
the file, and then enable System Integrity Protection

## Disable System Integrity Protection
Restart your system. Once the screen turns black, hold the command and R keys until the
Apple icon appears. This will boot your system into Recovery Mode.
Note: The slower than normal boot time is expected behavior.

Click on the Utilities menu at the top of the screen, and then click Terminal. Enter the following command:
```
csrutil disable && reboot
```

## Copy the PAM Module
When the computer has rebooted. Open a Terminal window, and run the following command:
```
sudo cp /usr/local/Cellar/pam_yubico/2.23/lib/security/pam_yubico.so
/usr/lib/pam/pam_yubico.so
```
Note: The version number 2.23 might change by time. Make sure to alter this to copy the file.

## Enable System Integrity Protection
Restart your system. Once the screen turns black, hold the command and R keys until the
Apple icon appears. This will boot your system into Recovery Mode.
Note: The slower than normal boot time is expected behavior.

Click on the Utilities menu at the top of the screen, and then click Terminal. Enter the following command:
```
csrutil enable && reboot
```

# Configure your Yubikey
Open the YubiKey Personalization Tool from your program folder on your Macbook and insert the YubiKey Neo in a USB port on your Mac.

1. Open the “Settings tab” at the top of the window, and ensure that the “Logging Settings”
section has logging enabled, and the “Yubico Output” selected.

2. Open the “Challenge Response” tab at the top of the window.

  1. Select Configuration Slot 2
  2. Select Variable input for HMAC-SHA1 Mode
  3. Click Generate to generate a new Secret Key (20 bytes Hex)
  4. Click Write Configuration

![Set Yubikey options](https://www.avisi.nl/assets/blog/wp-uploads/2014/03/yubico.jpg "YubiKey Personalization Tool")

Please note, you can do this with multiple Yubikeys as should you lose it!

You should configure both the Yubikey Neo and the Yubikey 4 with the Challenge-Response mode!

# Change Yubikey Mode
We also need to set the YubiKey mode to OTP/U2F/CCID. Open a Terminal and run the following command:

```
ykneomgr --set-mode=6
```

# Configure PAM on your Macbook
Open a Terminal window, and run the following command as your regular user, with firstly the Yubikey Neo inserted:

```
mkdir –m0700 –p ~/.yubico
ykpamcfg -2
```

Switch the Yubikey Neo to the Yubikey 4 and run the command again:

```
ykpamcfg -2
```

Both Yubikeys are now setup with your computer.


# Prepare GPG
We need a Memory disk on the Mac for when we generate a random key. A 20 megabyte memory disk can be created with the following command:

```
diskutil erasevolume HFS+ 'RAMDisk' `hdiutil attach -nomount ram://8388608`
```

Create a  new gpg.conf in the ram disk directory

```
$ cat << EOF > $GNUPGHOME/gpg.conf
use-agent
personal-cipher-preferences AES256 AES192 AES CAST5
personal-digest-preferences SHA512 SHA384 SHA256 SHA224
default-preference-list SHA512 SHA384 SHA256 SHA224 AES256 AES192 AES CAST5 ZLIB BZIP2 ZIP Uncompressed
cert-digest-algo SHA512
s2k-digest-algo SHA512
s2k-cipher-algo AES256
charset utf-8
fixed-list-mode
no-comments
no-emit-version
keyid-format 0xlong
list-options show-uid-validity
verify-options show-uid-validity
with-fingerprint
EOF
```

And make sure GPG starts using it:

```
export GNUPGHOME=$PWD
```

# Enable Yubikey for Auth, Sudo and Screensaver
Before you proceed, you should verify you have the `/usr/lib/pam/pam_yubico.so` file present on your Macbook from your preparations. If you dont, you will lock your self out of your Macbook!

Edit the following files:

* /etc/pam.d/authorization
* /etc/pam.d/sudo
* /etc/pam.d/screensaver

You need to use sudo to do so. From the terminal issue the following command:

```
sudo vi /etc/pam.d/screensaver
```

Add the following to the file:

```
auth required pam_yubico.so mode=challenge-response
```

Before you alter the `sudo` and `authorization` files, you can verify everything works by enabling the screensaver first. If you cannot login from the screensaver while the Yubikey is not present, something is terrible wrong now and you should NOT continue.

Use the screensaver to check both the Yubikeys before you proceed.

Also remember to set the screensaver to require password.

![Mac screensaver](https://www.avisi.nl/assets/blog/wp-uploads/2014/03/screensaver.jpg "Macbook Screensaver Password")

# Generating More Secure GPG Keys
Open a Terminal shell and start generating our new keys.

## Generating the Primary Key
Invoke `gpg --gen-key` with the `--expert` flag to expose some additional menu items.

```
$ gpg --expert --gen-key
gpg (GnuPG) 1.4.12; Copyright (C) 2012 Free Software Foundation, Inc.
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Please select what kind of key you want:
   (1) RSA and RSA (default)
   (2) DSA and Elgamal
   (3) DSA (sign only)
   (4) RSA (sign only)
   (7) DSA (set your own capabilities)
   (8) RSA (set your own capabilities)
Your selection? 8
```


# References
<https://www.yubico.com/wp-content/uploads/2015/04/YubiKey-OSX-Login.pdf>
<https://www.avisi.nl/blog/2014/05/06/two-factor-authentication-on-osx-a-yubikey-example/>
<https://spin.atomicobject.com/2013/11/24/secure-gpg-keys-guide/>
<https://medium.com/@ahawkins/securing-my-digital-life-gpg-yubikey-ssh-on-macos-5f115cb01266>
