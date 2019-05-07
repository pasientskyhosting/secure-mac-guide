This is a practical guide to using [YubiKey](https://www.yubico.com/faq/yubikey/) as a SmartCard for storing GPG encryption and signing keys. Keys stored on a SmartCard like YubiKey seem more difficult to steal than ones stored on disk, and are convenient for everyday use..

The blog "[Exploring Hard Tokens](https://www.avisi.nl/blog/2012/01/05/exploring-hard-tokens/)" describes the disadvantages of the combination of a username/password for access control. Passwords can be cracked or retrieved by social engineering. They can be read from faulty systems or even retrieved from unsecured internet access.

Authentication on a workstation often is done by using a username and password. Furthermore, it is almost impossible to detect when an attacker accesses a system. Therefore it is important to strengthen your authentication by adding a second step to your authentication process.

# Purchase YubiKey
We use the YubiKey 4 as it features 4096 bit keys.

You should also buy another YubiKey as a backup key for your computer login, because if you lose your YubiKey, you wont be able to login into your computer.

* https://www.yubico.com/products/yubikey-hardware/yubikey4/

# Prepare your Macbook

## Enable full disk encryption
Please make sure before you start this process, that your Macbook has enabled FileVault 2 disk encryption.
Apple has an excellent guide here https://support.apple.com/en-gb/HT204837

## Configure idle and hibernation
You may wish to enforce hibernation and evict FileVault keys from memory instead of traditional sleep to memory:

```
sudo pmset -a hibernatemode 25
sudo pmset -a destroyfvkeyonstandby 1
```

If you choose to evict FileVault keys in standby mode, you should also modify your standby and power nap settings. Otherwise, your machine may wake while in standby mode and then power off due to the absence of the FileVault key. See this [issue](https://github.com/drduh/macOS-Security-and-Privacy-Guide/issues/124) for more information. These settings can be changed with:

```
sudo pmset -a womp 0
sudo pmset -a powernap 0
sudo pmset -a standby 1
sudo pmset -a tcpkeepalive 0
sudo pmset -a ttyskeepawake 0
sudo pmset -a standbydelay 600
sudo pmset -a autopoweroff 1
sudo pmset -a autopoweroffdelay 605
```

## Enable Secure Keyboard Entry
Command line users who wish to add an additional layer of security to their keyboarding within Terminal app can find a helpful privacy feature built into the Mac client. Whether aiming for generally increasing security, if using a public Mac, or are simply concerned about things like keyloggers or any other potentially unauthorized access to your keystrokes and character entries, you can enable this feature in the Mac OS X Terminal app to secure keyboard entry and any command line input into the terminal.

### Mac builtin Terminal
Enable it for the build in Terminal on Macbook:

![SKI Terminal](http://cdn.osxdaily.com/wp-content/uploads/2011/12/secure-keyboard-entry.jpg "Terminal enable SKI")

### iTerm2
Enable it for iTerm which a lot of people use (highly recommended):

![SKI iTerm](https://mig5.net/sites/mig5.net/files/styles/medium/public/field/image/securekeyboard.png?itok=25YqK8AQ "iTerm enable SKI")

## Enable firewall and stealth mode
Built-in, basic firewall which blocks incoming connections only.
> Note: this firewall does not have the ability to monitor, nor block outgoing connections.

```
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setglobalstate on # enable fw
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setloggingmode on # enable logging
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setstealthmode on # dont respond to pings
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setallowsigned off
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --setallowsignedapp off
sudo pkill -HUP socketfilterfw
```

Computer hackers scan networks so they can attempt to identify computers to attack. When stealth mode is enabled, your computer does not respond to ICMP ping requests, and does not answer to connection attempts from a closed TCP or UDP port.

# Install required software
The required software for this guide is:

* Homebrew
* PAM Yubico
* YubiKey Personalization Tools
* GPG 2
* Knot-Resolver
* OpenSSL
* LibreSSL

## Install Homebrew
Open a Terminal window and then run the following command to install Homebrew:

```
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

## Install better packages
The version of OpenSSL in Sierra is 0.9.8zh which is not current. It doesn't support TLS 1.1 or newer, elliptic curve ciphers, and more.

Apple declares OpenSSL deprecated in their Cryptographic Services Guide document. Their version also has patches which may surprise you.

The version of Curl which comes with macOS uses Secure Transport for SSL/TLS validation.

```
brew install openssl
brew install libressl
brew install curl
brew install wget
```

### Configure your shell
To use LibreSSL and curl installed by Homebrew, it is important to update your path. You can add the following to your shell profile. Currently we're using zsh where the file you need to alter is `~/.zshrc`

Add the following to the file:

```
export PATH="/usr/local/opt/curl/bin:$PATH"
export PATH="/usr/local/opt/libressl/bin:$PATH"
```

## Install knot-resolver
Knot Resolver is an application that acts as a local DNS Privacy stub resolver (using DNS-over-TLS). Knot Resolver encrypts DNS queries sent from a client machine (desktop or laptop) to a DNS Privacy resolver increasing end user privacy.

```
brew install knot-resolver
```

You will also need some certificates so you're able to use DNS-over-TLS. To install the certificates run the following in a Terminal:

```
cd /usr/local/etc/kresd
wget https://secure.globalsign.net/cacert/Root-R2.crt
wget https://www.digicert.com/CACerts/DigiCertECCSecureServerCA.crt
openssl x509 -inform der -in Root-R2.crt -out GlobalSignR2CA.pem
openssl x509 -inform der -in DigiCertECCSecureServerCA.crt -out DigiCertECCSecureServerCA.pem
```

We then need to setup Knot to use DNS-Over-TLS.

Edit Knot Resolvers configuration file at `/usr/local/etc/kresd/config` and paste in the following content:

```
-- Listen on localhost (default)
net = { '127.0.0.1', '::1' }

-- Used for choosing random DNS Provider
require 'math'
math.randomseed(os.time())

-- Load Useful modules
modules = {
	'hints > iterate', -- Load /etc/hosts and allow custom root hints
	'stats',
	'predict',
   'policy',
   'serve_stale < cache',
   'workarounds < iterate',
}

-- Cache size
cache.size = 150 * MB

-- Prefetch learning (20-minute blocks over 72 hours)
predict.config({ window = 20, period = 72})

-- Randomize forwards DNS Queries
DigiCert_bundle='/usr/local/etc/kresd/DigiCertECCSecureServerCA.pem'
GlobalSign_bundle='/usr/local/etc/kresd/GlobalSignR2CA.pem'

dns_providers = {
  {
    {'1.1.1.1', hostname='cloudflare-dns.com', ca_file=DigiCert_bundle},
    {'1.0.0.1', hostname='cloudflare-dns.com', ca_file=DigiCert_bundle},
    {'2606:4700:4700::1111', hostname='cloudflare-dns.com', ca_file=DigiCert_bundle},
    {'2606:4700:4700::1001', hostname='cloudflare-dns.com', ca_file=DigiCert_bundle},
  },
  {
    {'9.9.9.9', hostname='dns.quad9.net', ca_file=DigiCert_bundle},
    {'149.112.112.112', hostname='dns.quad9.net', ca_file=DigiCert_bundle},
    {'2620:fe::fe', hostname='dns.quad9.net', ca_file=DigiCert_bundle},
    {'2620:fe::9', hostname='dns.quad9.net', ca_file=DigiCert_bundle},
  },
  {
    {'8.8.8.8', hostname='dns.google', ca_file=GlobalSign_bundle},
    {'8.8.4.4', hostname='dns.google', ca_file=GlobalSign_bundle},
    {'2001:4860:4860::8888', hostname='dns.google', ca_file=GlobalSign_bundle},
    {'2001:4860:4860::8844', hostname='dns.google', ca_file=GlobalSign_bundle},
  }
}

tls_forwarders = {}
for n, fwdspec in ipairs(dns_providers) do
  table.insert(tls_forwarders, policy.TLS_FORWARD(fwdspec))
end

policy.add(function (request, query)
  return tls_forwarders[math.random(1, #tls_forwarders)]
end)
```

Enable Knot Resolver to start at boot:

```
sudo brew services start knot-resolver
```

Then enable Knot Resolver DNS for each interface on your Mac:

```
networksetup -listallnetworkservices 2>/dev/null | grep -v '*' | while read x ; do
    networksetup -setdnsservers "$x" 127.0.0.1 ::1
done
```

### Block DNS queries
You should block all connections to other DNS servers as various programs use some sort of internal DNS resolver. Chrome has this build in, lots of programs also falls back to systemd's resolver. So to make sure we always use Stubby as DNS resolver, we simply just block all DNS connections to anything but Knot Resolver:

Start of by editing `/etc/pf.conf` and add the following line to the **end** of the file:

```
block drop quick on !lo0 proto udp from any to any port = 53
```

Then reload the firewall with:

```
sudo pfctl -ef /etc/pf.conf
```

Verify that the rule is active with:
```
pfctl -v -s rules
```

### Test Knot Resolver
A quick test can be done by using dig (or your favorite DNS tool) on the loopback address

```
dig @127.0.0.1 www.example.com

; <<>> DiG 9.9.7-P3 <<>> @127.0.0.1 www.example.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 52807
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 2, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; OPT=8: 00 00 00 00  (.) (.) (.) (.)
;; QUESTION SECTION:
;www.example.com.		IN	A

;; ANSWER SECTION:
wWW.ExAmPLe.com.	27319	IN	A	93.184.216.34

;; AUTHORITY SECTION:
ExAmPLe.com.		1751	IN	NS	b.iana-servers.net.
ExAmPLe.com.		1751	IN	NS	a.iana-servers.net.

;; Query time: 226 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)
;; WHEN: Mon Oct 30 09:56:58 CET 2017
;; MSG SIZE  rcvd: 169
```

You should also test and make sure you cannot use external DNS servers. The following should give you a timeout:

```
dig @8.8.8.8 www.example.com

; <<>> DiG 9.9.7-P3 <<>> @8.8.8.8 www.example.com
; (1 server found)
;; global options: +cmd
;; connection timed out; no servers could be reached
```

## Install GPG
Open a Terminal window and then run the following commands:

```
brew install gnupg2 pinentry-mac coreutils
```

## Install YubiKey Personalization Tools
Install the latest version of the YubiKey Personalization Tool from the App Store
https://itunes.apple.com/us/app/yubikey-personalization-tool/id638161122?mt=12

# Configure your YubiKey
Open the YubiKey Personalization Tool from your program folder on your Macbook and insert the YubiKey in a USB port on your Mac.

1. Open the "Settings tab at the top of the window, and ensure that the "Logging Settings"
section has logging enabled, and the “Yubico Format“ selected.

2. Open the “Challenge Response” tab at the top of the window. Then configure slot 2:

  1. Select Configuration Slot 2
  2. Select Variable input for HMAC-SHA1 Mode
  3. Click Generate to generate a new Secret Key (20 bytes Hex)
  4. Make sure it the box is `unchecked` for "Require user input (button press)"
  5. Click Write Configuration

![Set Yubikey options](https://crewjam.com/images/YubiKey_Personalization_Tool_and_MacOS_X_Challenge-Response.png "YubiKey Personalization Tool")

You must configure both the YubiKeys with the Challenge-Response mode now.

# Install PAM Yubico
Open a Terminal window, and run the following command:
```
brew install pam_yubico
```

## Configure PAM on your Macbook
Open a Terminal window, and run the following command as your regular user, with firstly the YubiKey inserted.

> Note: If you have secure keyboard input enabled for your terminal, this will give an error. Disable while you run the commands and reenable it.

```
mkdir –p ~/.yubico
chmod -R 0700 ~/.yubico
ykpamcfg -2
```

Your YubiKey are now setup with your Macbook and can be used. You should store the backup YubiKey somewhere safe for recovery - like in a vault in your bank ;)

## Enable YubiKey for Auth, Sudo and Screensaver
Before you proceed, you should verify you have the `/usr/local/lib/security/pam_yubico.so` file present on your Macbook from your earlier preparations. If you dont, you will lock your self out of your Macbook now.

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
auth       required       /usr/local/lib/security/pam_yubico.so mode=challenge-response
```

Ending up with something like this

```
auth       optional       pam_krb5.so use_first_pass use_kcminit
auth       required       pam_opendirectory.so use_first_pass nullok
auth       required       /usr/local/lib/security/pam_yubico.so mode=challenge-response
account    required       pam_opendirectory.so
account    sufficient     pam_self.so
account    required       pam_group.so no_warn group=admin,wheel fail_safe
account    required       pam_group.so no_warn deny group=admin,wheel ruser fail_safe
```

Also remember to set the screensaver to require password or it wont work anyway :)

![Mac screensaver](https://i.stack.imgur.com/BwMhk.png "Macbook Screensaver Password")

Before you alter the `sudo` and `authorization` files, you can verify everything works by enabling the screensaver first. If you cannot login from the screensaver while the YubiKey is present, something is terrible wrong now and you should NOT continue.

Use the screensaver to check both the YubiKeys before you proceed.

# Enable Yubikeylockd
Yubikeylockd is a simple daemon that locks your computer (starts the screensaver) when you unplug the YubiKey. This is ideal for when you leave the computer and you simply just take the YubiKey out and it will simply lock automatically.

To install Yubikeylockd, open a Terminal and enter the following command:

```
brew install https://raw.githubusercontent.com/shtirlic/yubikeylockd/master/yubikeylockd.rb
```

When brew is done, you need to enable the service. Enter the following command:

```
sudo brew services start yubikeylockd
```

## Disable yubikeylockd
If you somehow do not want to have Yubikeylockd enabled anymore, yet we wouldn't recommend it. Open a Terminal and enter the following command:

```
sudo brew services stop yubikeylockd
```

# Prepare GPG
We need a RAM disk on the Mac for when we generate a random key. A 4 gigabyte RAM disk can be created with the following command:

```
diskutil erasevolume HFS+ 'RAMDisk' `hdiutil attach -nomount ram://8388608`
```

Create a new `gpg.conf` in the ram disk directory

```
cat << EOF > /Volumes/RAMDisk/gpg.conf
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

And make sure GPG starts using it and language is English and we have set the right permissions on files. Do not close the Terminal after this, as you need the exported variables present in your shell.

```
export GNUPGHOME=/Volumes/RAMDisk
export LANG=en
umask 070
```

## Generating the Primary Key
Use the `same terminal session` you ran the export of GNUPGHOME in when continuing the next steps.

Generate a new key with GPG, selecting RSA (sign only) and the appropriate keysize, optionally specifying an expiry:

```
gpg --full-generate-key

Please select what kind of key you want:
   (1) RSA and RSA (default)
   (2) DSA and Elgamal
   (3) DSA (sign only)
   (4) RSA (sign only)
Your selection? 4

RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (2048) 4096

Requested keysize is 4096 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years

Key is valid for? (0) 0
Key does not expire at all

Is this correct? (y/N) y

You need a user ID to identify your key; the software constructs the user ID
from the Real Name, Comment and Email Address in this form:
    "Heinrich Heine (Der Dichter) <heinrichh@duesseldorf.de>"

Real name: Dr Duh
Email address: doc@duh.to
Comment:
You selected this USER-ID:
    "Dr Duh <doc@duh.to>"

Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? o
You need a Passphrase to protect your secret key.

gpg: key 0xFF3E7D88647EBCDB marked as ultimately trusted
public and secret key created and signed.

gpg: checking the trustdb
gpg: 3 marginal(s) needed, 1 complete(s) needed, PGP trust model
gpg: depth: 0  valid:   1  signed:   0  trust: 0-, 0q, 0n, 0m, 0f, 1u
pub   4096R/0xFF3E7D88647EBCDB 2016-05-24
      Key fingerprint = 011C E16B D45B 27A5 5BA8  776D FF3E 7D88 647E BCDB
uid                 [ultimate] Dr Duh <doc@duh.to>

Note that this key cannot be used for encryption.  You may want to use
the command "--edit-key" to generate a subkey for this purpose.
```

### Save Key ID
Export the key ID as a variable for use throughout:

```
export KEYID=0xFF3E7D88647EBCDB
```

This step is important. You can see the key id from one of the last lines in the output from above:

```
pub   4096R/0xFF3E7D88647EBCDB 2016-05-24
```

### Create revocation certificate
Create a way to revoke your keys in case of loss or compromise, an explicit reason being optional

```
gpg --gen-revoke $KEYID > $GNUPGHOME/revoke.txt

sec  4096R/0xFF3E7D88647EBCDB 2016-05-24 Dr Duh <doc@duh.to>

Create a revocation certificate for this key? (y/N) y

Please select the reason for the revocation:
  0 = No reason specified
  1 = Key has been compromised
  2 = Key is superseded
  3 = Key is no longer used
  Q = Cancel
(Probably you want to select 1 here)
Your decision? 1

Enter an optional description; end it with an empty line:
> (Press enter)

Reason for revocation: Key has been compromised
(No description given)

Is this okay? (y/N) y

You need a passphrase to unlock the secret key for
user: "Dr Duh <doc@duh.to>"
4096-bit RSA key, ID 0xFF3E7D88647EBCDB, created 2016-05-24

ASCII armored output forced.
Revocation certificate created.

Please move it to a medium which you can hide away; if Mallory gets
access to this certificate he can use it to make your key unusable.
It is smart to print this certificate and store it away, just in case
your media become unreadable.  But have some caution:  The print system of
your machine might store the data and make it available to others!
```

### Back up master key
Save a copy of the private key block:

```
gpg --armor --export-secret-keys $KEYID > $GNUPGHOME/master.key
```

### Create subkeys
Edit the key to add subkeys:

```
gpg --expert --edit-key $KEYID

Secret key is available.

pub  4096R/0xFF3E7D88647EBCDB  created: 2016-05-24  expires: never       usage: SC
                               trust: ultimate      validity: ultimate
[ultimate] (1). Dr Duh <doc@duh.to>
```

Now lets create the keys

#### Signing key
```
gpg> addkey
Key is protected.

You need a passphrase to unlock the secret key for
user: "Dr Duh <doc@duh.to>"
4096-bit RSA key, ID 0xFF3E7D88647EBCDB, created 2016-05-24

Please select what kind of key you want:
   (3) DSA (sign only)
   (4) RSA (sign only)
   (5) Elgamal (encrypt only)
   (6) RSA (encrypt only)
   (7) DSA (set your own capabilities)
   (8) RSA (set your own capabilities)
  (10) ECC (sign only)
  (11) ECC (set your own capabilities)
  (12) ECC (encrypt only)
  (13) Existing key
Your selection? 4

RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (2048) 4096
Requested keysize is 4096 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 0
Key does not expire at all
Is this correct? (y/N) y
Really create? (y/N) y
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.

...................+++++
..+++++

pub  4096R/0xFF3E7D88647EBCDB  created: 2016-05-24  expires: never       usage: SC
                               trust: ultimate      validity: ultimate
sub  4096R/0xBECFA3C1AE191D15  created: 2016-05-24  expires: never       usage: S
[ultimate] (1). Dr Duh <doc@duh.to>
```

#### Encryption key
```
gpg> addkey
Key is protected.

You need a passphrase to unlock the secret key for
user: "Dr Duh <doc@duh.to>"
4096-bit RSA key, ID 0xFF3E7D88647EBCDB, created 2016-05-24

Please select what kind of key you want:
   (3) DSA (sign only)
   (4) RSA (sign only)
   (5) Elgamal (encrypt only)
   (6) RSA (encrypt only)
   (7) DSA (set your own capabilities)
   (8) RSA (set your own capabilities)
  (10) ECC (sign only)
  (11) ECC (set your own capabilities)
  (12) ECC (encrypt only)
  (13) Existing key
Your selection? 6

RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (2048) 4096
Requested keysize is 4096 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 0
Key does not expire at all
Is this correct? (y/N) y
Really create? (y/N) y
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.

.+++++
...........+++++

pub  4096R/0xFF3E7D88647EBCDB  created: 2016-05-24  expires: never       usage: SC
                               trust: ultimate      validity: ultimate
sub  4096R/0xBECFA3C1AE191D15  created: 2016-05-24  expires: never       usage: S
sub  4096R/0x5912A795E90DD2CF  created: 2016-05-24  expires: never       usage: E
[ultimate] (1). Dr Duh <doc@duh.to>
```

#### Authentication key
This is the important key in our guide. This key is used for SSH authentication

```
gpg> addkey
Key is protected.

You need a passphrase to unlock the secret key for
user: "Dr Duh <doc@duh.to>"
4096-bit RSA key, ID 0xFF3E7D88647EBCDB, created 2016-05-24

Please select what kind of key you want:
   (3) DSA (sign only)
   (4) RSA (sign only)
   (5) Elgamal (encrypt only)
   (6) RSA (encrypt only)
   (7) DSA (set your own capabilities)
   (8) RSA (set your own capabilities)
  (10) ECC (sign only)
  (11) ECC (set your own capabilities)
  (12) ECC (encrypt only)
  (13) Existing key

Your selection? 8

Possible actions for a RSA key: Sign Encrypt Authenticate
Current allowed actions: Sign Encrypt

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? s

Possible actions for a RSA key: Sign Encrypt Authenticate
Current allowed actions: Encrypt

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? e

Possible actions for a RSA key: Sign Encrypt Authenticate
Current allowed actions:

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? a

Possible actions for a RSA key: Sign Encrypt Authenticate
Current allowed actions: Authenticate

   (S) Toggle the sign capability
   (E) Toggle the encrypt capability
   (A) Toggle the authenticate capability
   (Q) Finished

Your selection? q
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (2048) 4096
Requested keysize is 4096 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 0
Key does not expire at all
Is this correct? (y/N) y
Really create? (y/N) y
We need to generate a lot of random bytes. It is a good idea to perform
some other action (type on the keyboard, move the mouse, utilize the
disks) during the prime generation; this gives the random number
generator a better chance to gain enough entropy.

+++++
.....+++++

pub  4096R/0xFF3E7D88647EBCDB  created: 2016-05-24  expires: never       usage: SC
                               trust: ultimate      validity: ultimate
sub  4096R/0xBECFA3C1AE191D15  created: 2016-05-24  expires: never       usage: S
sub  4096R/0x5912A795E90DD2CF  created: 2016-05-24  expires: never       usage: E
sub  4096R/0x3F29127E79649A3D  created: 2016-05-24  expires: never       usage: A
[ultimate] (1). Dr Duh <doc@duh.to>
```

Then save your work

```
gpg> save
```

### Check your work
List your new secret keys:

```
gpg --list-secret-keys

/Volumes/RAMDisk/pubring.kbx
-------------------------------
sec   4096R/0xFF3E7D88647EBCDB 2016-05-24
      Key fingerprint = 011C E16B D45B 27A5 5BA8  776D FF3E 7D88 647E BCDB
uid                            Dr Duh <doc@duh.to>
ssb   2048R/0xBECFA3C1AE191D15 2016-05-24
ssb   2048R/0x5912A795E90DD2CF 2016-05-24
ssb   2048R/0x3F29127E79649A3D 2016-05-24
```

### Export subkeys
Save a copy of your subkeys:
```
gpg --armor --export-secret-keys $KEYID > $GNUPGHOME/mastersub.key
gpg --armor --export-secret-subkeys $KEYID > $GNUPGHOME/sub.key
```

### Export public keys
This file should be publicly shared:

```
gpg --armor --export $KEYID > $HOME/pubkey.txt
```

Optionally, it may be uploaded to a public keyserver:

```
gpg --keyserver pgp.mit.edu --send-key $KEYID
```

After a little while, it ought to propagate to other servers.

### Back up everything
Once keys are moved to hardware, they cannot be extracted again (otherwise, what would be the point?), so make sure you have made an encrypted backup before proceeding.

We recommend to back up the keys to an encrypted USB device. You will need this backup in case you LOSE your YubiKey and need to create a new one

# Configure YubiKey as SmartCard
Plug in your YubiKey and enter the following command in a Terminal:

```
gpg --card-edit

Reader ...........: Yubico Yubikey 4 OTP U2F CCID
Application ID ...: D2760001240102010006056699490000
Version ..........: 2.1
Manufacturer .....: Yubico
Serial number ....: 05669949
Name of cardholder: [ikke indstillet]
Language prefs ...: [ikke indstillet]
Sex ..............: ikke angivet
URL of public key : [ikke indstillet]
Login data .......: [ikke indstillet]
Signature PIN ....: tvunget
Key attributes ...: rsa2048 rsa2048 rsa2048
Max. PIN lengths .: 127 127 127
PIN retry counter : 3 3 3
Signature counter : 0
Signature key ....: [none]
Encryption key....: [none]
Authentication key: [none]
General key info..: [none]
```

## Change PINs
The default PIN codes are `12345678` for admin and `123456` for default use. You will need to use the default pin on everyday basis when you need to use the ssh key for auth. The Admin key is only if you want to alter data on the card.

Do not lose these pins EVER or you'll have to reset the card with the included `yubikey-reset.sh` script in this repo.

```
gpg/card> admin
Admin commands are allowed

gpg/card> passwd
gpg: OpenPGP card no. D2760001240102010006055532110000 detected

1 - change PIN
2 - unblock PIN
3 - change Admin PIN
4 - set the Reset Code
Q - quit

Your selection? 3
PIN changed.

1 - change PIN
2 - unblock PIN
3 - change Admin PIN
4 - set the Reset Code
Q - quit

1 - change PIN
2 - unblock PIN
3 - change Admin PIN
4 - set the Reset Code
Q - quit

Your selection? 1
PIN changed.

1 - change PIN
2 - unblock PIN
3 - change Admin PIN
4 - set the Reset Code
Q - quit

Your selection? q
```

## Set card information
```
gpg/card> name
Cardholder's surname: Duh
Cardholder's given name: Dr

gpg/card> lang
Language preferences: en

gpg/card> login
Login data (account name): doc@duh.to

gpg/card> sex
Sex ((M)ale, (F)emale or space): m

gpg/card> quit
```

## Transfer keys
```
gpg --edit-key $KEYID

Secret key is available.

pub  4096R/0xFF3E7D88647EBCDB  created: 2016-05-24  expires: never       usage: SC
                               trust: ultimate      validity: ultimate
sub  4096R/0xBECFA3C1AE191D15  created: 2016-05-24  expires: never       usage: S
sub  4096R/0x5912A795E90DD2CF  created: 2016-05-24  expires: never       usage: E
sub  4096R/0x3F29127E79649A3D  created: 2016-05-24  expires: never       usage: A
[ultimate] (1). Dr Duh <doc@duh.to>

gpg> toggle

sec  4096R/0xFF3E7D88647EBCDB  created: 2016-05-24  expires: never
ssb  4096R/0xBECFA3C1AE191D15  created: 2016-05-24  expires: never
ssb  4096R/0x5912A795E90DD2CF  created: 2016-05-24  expires: never
ssb  4096R/0x3F29127E79649A3D  created: 2016-05-24  expires: never
(1)  Dr Duh <doc@duh.to>
```

### Signature key
Move the signature key (you will be prompted for the key passphrase and admin PIN):

```
gpg> key 1

sec  4096R/0xFF3E7D88647EBCDB  created: 2016-05-24  expires: never
ssb* 4096R/0xBECFA3C1AE191D15  created: 2016-05-24  expires: never
ssb  4096R/0x5912A795E90DD2CF  created: 2016-05-24  expires: never
ssb  4096R/0x3F29127E79649A3D  created: 2016-05-24  expires: never
(1)  Dr Duh <doc@duh.to>

gpg> keytocard
Signature key ....: [none]
Encryption key....: [none]
Authentication key: [none]

Please select where to store the key:
   (1) Signature key
   (3) Authentication key
Your selection? 1

You need a passphrase to unlock the secret key for
user: "Dr Duh <doc@duh.to>"
4096-bit RSA key, ID 0xBECFA3C1AE191D15, created 2016-05-24


sec  4096R/0xFF3E7D88647EBCDB  created: 2016-05-24  expires: never
ssb* 4096R/0xBECFA3C1AE191D15  created: 2016-05-24  expires: never
                     card-no: 0006 05553211
ssb  4096R/0x5912A795E90DD2CF  created: 2016-05-24  expires: never
ssb  4096R/0x3F29127E79649A3D  created: 2016-05-24  expires: never
(1)  Dr Duh <doc@duh.to>
```

### Encryption key
Type key 1 again to deselect and key 2 to select the next key:
```
gpg> key 1

sec  4096R/0xFF3E7D88647EBCDB  created: 2016-05-24  expires: never
ssb  4096R/0xBECFA3C1AE191D15  created: 2016-05-24  expires: never
                     card-no: 0006 05553211
ssb  4096R/0x5912A795E90DD2CF  created: 2016-05-24  expires: never
ssb  4096R/0x3F29127E79649A3D  created: 2016-05-24  expires: never
(1)  Dr Duh <doc@duh.to>

gpg> key 2

sec  4096R/0xFF3E7D88647EBCDB  created: 2016-05-24  expires: never
ssb  4096R/0xBECFA3C1AE191D15  created: 2016-05-24  expires: never
                     card-no: 0006 05553211
ssb* 4096R/0x5912A795E90DD2CF  created: 2016-05-24  expires: never
ssb  4096R/0x3F29127E79649A3D  created: 2016-05-24  expires: never
(1)  Dr Duh <doc@duh.to>
```

Move the encryption key to card:

```
gpg> keytocard
Signature key ....: 07AA 7735 E502 C5EB E09E  B8B0 BECF A3C1 AE19 1D15
Encryption key....: [none]
Authentication key: [none]

Please select where to store the key:
   (2) Encryption key
Your selection? 2

You need a passphrase to unlock the secret key for
user: "Dr Duh <doc@duh.to>"
4096-bit RSA key, ID 0x5912A795E90DD2CF, created 2016-05-24

sec  4096R/0xFF3E7D88647EBCDB  created: 2016-05-24  expires: never
ssb  4096R/0xBECFA3C1AE191D15  created: 2016-05-24  expires: never
                     card-no: 0006 05553211
ssb* 4096R/0x5912A795E90DD2CF  created: 2016-05-24  expires: never
                     card-no: 0006 05553211
ssb  4096R/0x3F29127E79649A3D  created: 2016-05-24  expires: never
(1)  Dr Duh <doc@duh.to>
```

### Authentication key
Type key 2 again to deselect and key 3 to select the next key:

```
gpg> key 2

sec  4096R/0xFF3E7D88647EBCDB  created: 2016-05-24  expires: never
ssb  4096R/0xBECFA3C1AE191D15  created: 2016-05-24  expires: never
                     card-no: 0006 05553211
ssb  4096R/0x5912A795E90DD2CF  created: 2016-05-24  expires: never
                     card-no: 0006 05553211
ssb  4096R/0x3F29127E79649A3D  created: 2016-05-24  expires: never
(1)  Dr Duh <doc@duh.to>

gpg> key 3

sec  4096R/0xFF3E7D88647EBCDB  created: 2016-05-24  expires: never
ssb  4096R/0xBECFA3C1AE191D15  created: 2016-05-24  expires: never
                     card-no: 0006 05553211
ssb  4096R/0x5912A795E90DD2CF  created: 2016-05-24  expires: never
                     card-no: 0006 05553211
ssb* 4096R/0x3F29127E79649A3D  created: 2016-05-24  expires: never
(1)  Dr Duh <doc@duh.to>

gpg> keytocard
Signature key ....: 07AA 7735 E502 C5EB E09E  B8B0 BECF A3C1 AE19 1D15
Encryption key....: 6F26 6F46 845B BEB8 BDF3  7E9B 5912 A795 E90D D2CF
Authentication key: [none]

Please select where to store the key:
   (3) Authentication key
Your selection? 3

You need a passphrase to unlock the secret key for
user: "Dr Duh <doc@duh.to>"
4096-bit RSA key, ID 0x3F29127E79649A3D, created 2016-05-24

sec  4096R/0xFF3E7D88647EBCDB  created: 2016-05-24  expires: never
ssb  4096R/0xBECFA3C1AE191D15  created: 2016-05-24  expires: never
                     card-no: 0006 05553211
ssb  4096R/0x5912A795E90DD2CF  created: 2016-05-24  expires: never
                     card-no: 0006 05553211
ssb* 4096R/0x3F29127E79649A3D  created: 2016-05-24  expires: never
                     card-no: 0006 05553211
(1)  Dr Duh <doc@duh.to>
```

Save and quit:

```
gpg> save
```

## Securely cleanup
When you are done, and you have made a backup of your work to an ENCRYPTED usb-drive, issue a secure erase of the RAM disk:

```
gshred -zun 12 /Volumes/RAMDisk/*
```

And unmount the RAM disk again

```
diskutil unmountDisk force /Volumes/RAMDisk
```

When you have erased and unmounted the RAM disk, reboot your Macbook and you are all set with a more secure Macbook.

# Using the Keys on your Macbook
Start by opening a new terminal session and paste the following text into a terminal window to create a recommended GPG configuration:
```
mkdir -p ~/.gnupg

cat << EOF > ~/.gnupg/gpg.conf
auto-key-locate keyserver
keyserver hkps://hkps.pool.sks-keyservers.net
personal-cipher-preferences AES256 AES192 AES CAST5
personal-digest-preferences SHA512 SHA384 SHA256 SHA224
default-preference-list SHA512 SHA384 SHA256 SHA224 AES256 AES192 AES CAST5 ZLIB BZIP2 ZIP Uncompressed
cert-digest-algo SHA512
s2k-cipher-algo AES256
s2k-digest-algo SHA512
charset utf-8
fixed-list-mode
no-comments
no-emit-version
keyid-format 0xlong
list-options show-uid-validity
verify-options show-uid-validity
with-fingerprint
use-agent
require-cross-certification
EOF

cat << EOF > ~/.gnupg/gpg-agent.conf
enable-ssh-support
pinentry-program /usr/local/bin/pinentry-mac
default-cache-ttl 10800
max-cache-ttl 10800
EOF
```

## Import public key into your keyring
Import it from a file:

```
gpg --import < $HOME/pubkey.txt
gpg: key 0xFF3E7D88647EBCDB: public key "Dr Duh <doc@duh.to>" imported
gpg: Total number processed: 1
gpg:               imported: 1  (RSA: 1)
```

## Trust master key
Edit the imported key to assign it ultimate trust:
```
gpg --edit-key 0xFF3E7D88647EBCDB

Secret key is available.

pub  4096R/0xFF3E7D88647EBCDB  created: 2016-05-24  expires: never       usage: SC
                               trust: unknown       validity: unknown
sub  4096R/0xBECFA3C1AE191D15  created: 2016-05-24  expires: never       usage: S
sub  4096R/0x5912A795E90DD2CF  created: 2016-05-24  expires: never       usage: E
sub  4096R/0x3F29127E79649A3D  created: 2016-05-24  expires: never       usage: A
[ unknown] (1). Dr Duh <doc@duh.to>

gpg> trust
pub  4096R/0xFF3E7D88647EBCDB  created: 2016-05-24  expires: never       usage: SC
                               trust: unknown       validity: unknown
sub  4096R/0xBECFA3C1AE191D15  created: 2016-05-24  expires: never       usage: S
sub  4096R/0x5912A795E90DD2CF  created: 2016-05-24  expires: never       usage: E
sub  4096R/0x3F29127E79649A3D  created: 2016-05-24  expires: never       usage: A
[ unknown] (1). Dr Duh <doc@duh.to>

Please decide how far you trust this user to correctly verify other users' keys
(by looking at passports, checking fingerprints from different sources, etc.)

  1 = I don't know or won't say
  2 = I do NOT trust
  3 = I trust marginally
  4 = I trust fully
  5 = I trust ultimately
  m = back to the main menu

Your decision? 5
Do you really want to set this key to ultimate trust? (y/N) y

pub  4096R/0xFF3E7D88647EBCDB  created: 2016-05-24  expires: never       usage: SC
                               trust: ultimate      validity: unknown
sub  4096R/0xBECFA3C1AE191D15  created: 2016-05-24  expires: never       usage: S
sub  4096R/0x5912A795E90DD2CF  created: 2016-05-24  expires: never       usage: E
sub  4096R/0x3F29127E79649A3D  created: 2016-05-24  expires: never       usage: A
[ unknown] (1). Dr Duh <doc@duh.to>
Please note that the shown key validity is not necessarily correct
unless you restart the program.

gpg> quit
```

## Update your Shell Environment
For pretty much all shells. I use `zsh`, so i alter the `~/.zshrc` file:

```
export GPG_TTY=$(tty)
export SSH_AUTH_SOCK=$(gpgconf --list-dirs agent-ssh-socket);
```

## Restart
To make changes take affect, please restart the GPG agent `in a new terminal where GPG_TTY and SSH_AUTH_SOCK is set`:

```
gpg-connect-agent killagent /bye
gpg-connect-agent /bye
```

## Verify your work
There is a -L option of ssh-add that lists public key parameters of all identities currently represented by the agent. Copy and paste the following output to the server authorized_keys file:

```
ssh-add -L
ssh-rsa AAAAB4NzaC1yc2EAAAADAQABAAACAz[...]zreOKM+HwpkHzcy9DQcVG2Nw== cardno:000605553211
```

If you see a SSH key with the `cardno:` descriptions, you have now successfully setup a SSH key on your YubiKey.

You can now copy this public key to the servers you want to use it on etc.

# Misc
Different information and help.

## Loss of Yubikey
In case you lose your YubiKey, everything is not yet over and data is not yet lost. If you have another YubiKey nearby, you can simply redeploy the secure keys to a new YubiKey.

### Access to private SSH key
Start by recreating the RAMDisk drive with hdutil as done when you created the keys. Next, copy back all the files from your secure backup you took, to the RAMDisk.
When files are in place, run the export commands again to set the GNUPGHOME folder to the RAMDisk. Next, list your keys with keygrip:

```
gpg -k --with-keygrip

/Volumes/RAMDisk/pubring.kbx
------------------------------
pub   rsa2048/93BDD96B 2017-06-29 [SC]
      D03833D3D52F5FFCCC73452461671825E8DEC139
      Keygrip = 8A6CDC5FCE05A5B251BD8C397B269607534B4702
uid         [ultimate] Big John <big.john@gmail.com>
sub   rsa2048/0424163D 2017-06-29 [E]
      Keygrip = E110250E32B811D45879A66F487CE95BC1906D77
sub   rsa2048/8F228EDB 2017-06-29 [A]
      Keygrip = 32BC5688805A640D495E8A7B41EC78F74E77E098
```

Grab the unique id from the Authentication key which is the key with the [A] next to it. Add that keygrip to GNUPG's sshcontrol:

```
echo 32BC5688805A640D495E8A7B41EC78F74E77E098 > /Volumes/RAMDisk/sshcontrol
```

You need to copy the gpg-agent config to your RAMDisk too:

```
cp ~/.gnupg/gpg-agent.conf /Volumes/RAMDisk
```

Then export the SSH_AUTH_SOCK to point to the RAMDisk instead and restart your gpg agent:

```
export "SSH_AUTH_SOCK=/Volumes/RAMDisk/S.gpg-agent.ssh"

gpg-connect-agent killagent /bye
gpg-connect-agent /bye
```

You can verify that SSH is now able to see the private key from your keyring by issuing `ssh-add -L`

## Requiring touch to authenticate
By default the YubiKey will perform key operations without requiring a touch from the user. To require a touch for every SSH connection, use the [YubiKey Manager](https://developers.yubico.com/yubikey-manager/) (you'll need the Admin PIN):

```
ykman openpgp touch aut on
```

To require a touch for the signing and encrypting keys as well:

```
ykman openpgp touch sig on
ykman openpgp touch enc on
```

The YubiKey will blink when it's waiting for the touch.

## Change YubiKey Mode
We also need to set the YubiKey mode to OTP/U2F/CCID. My YubiKey 4 was pr. default in this mode. See more at https://developers.yubico.com/libu2f-host/Mode_switch_YubiKey.html

If you prefer a GUI, install the YubiKey Neo Manager: https://developers.yubico.com/yubikey-neo-manager - works with YubiKey 4 too.

# References
* https://www.yubico.com/wp-content/uploads/2015/04/YubiKey-OSX-Login.pdf
* https://www.avisi.nl/blog/2014/05/06/two-factor-authentication-on-osx-a-yubikey-example/
* https://spin.atomicobject.com/2013/11/24/secure-gpg-keys-guide/
* https://medium.com/@ahawkins/securing-my-digital-life-gpg-yubikey-ssh-on-macos-5f115cb01266
* https://github.com/drduh/YubiKey-Guide
* https://florin.myip.org/blog/easy-multifactor-authentication-ssh-using-yubikey-neo-tokens
* https://gist.github.com/bcomnes/647477a3a143774069755d672cb395ca
* https://wiki.archlinux.org/index.php/GnuPG#pinentry
* https://stackoverflow.com/questions/33961302/change-the-language-of-gnupg-on-a-mac
* https://gpgtools.tenderapp.com/kb/gpg-keychain-faq/how-to-move-secret-keys-to-usb-drive
* https://github.com/shtirlic/yubikeylockd
* https://getdnsapi.net/blog/dns-privacy-daemon-stubby/
* https://dnsprivacy.org/wiki/pages/viewpage.action?pageId=3145812
* https://derflounder.wordpress.com/2011/11/23/using-the-command-line-to-unlock-or-decrypt-your-filevault-2-encrypted-boot-drive/
