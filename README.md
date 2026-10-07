# John the Ripper — Kali Linux Lab

This guide explains how passwords are stored, what is **John the Ripper** and showing how I installed it on Kali, ran into a problem, and fixed it by building a newer version from source.

## Contents

- [0. How Passwords Are Stored](#0-how-passwords-are-stored)
- [1. Install John](#1-install-john)
- [2. Prepare the Hash](#2-prepare-the-hash)
- [3. The yescrypt Problem](#3-the-yescrypt-problem)
- [5. Build a New Version](#5-build-a-new-version)
- [6. Use RockYou](#6-use-rockyou)
- [7. Run John Again](#7-run-john-again)
- [8. Useful Commands](#8-useful-commands)

---

## 0. How Passwords Are Stored

Passwords are usually not stored as plain text. Instead, they are stored as hashes, which are one-way representations of the original password. For example, a password SHA-256 hash might look like: `$y$j9T$5f7K3mQx8Wz2LrNp$QvX9mN8kR3sT6pY2aBcD4eF7gH1jK5lM9nP2qR6sT8uV`

**_John the Ripper_** is a open-source password-cracking tool that allows us to test password strength. Its goal is simple: take a list of password hashes and find the original plaintext passwords by hashing guesses and and compares it with the target hash. If they match, John has found the original password.

---

## 1. Install John

By default, this tool should be included in Kali's toolset, but in my case, it wasn’t.
You can check whether it is installed with: `which john`.
I installed John using the Kali package manager:

```bash
sudo apt update
sudo apt install john
```

Then checked the version:

```bash
john
```

I got: `John the Ripper 1.9.0-jumbo-1`

---

## 2. Prepare the Hash

Before using John, we need the contents of `/etc/passwd` and `etc/shadow` in text files. `/etc/passwd` contains user information, while `/etc/shadow` contains password hashes. `unshadow` command combines both files into a format that **John the Ripper** can properly process:

```bash
sudo unshadow /etc/passwd /etc/shadow > ~/hashes.txt
```

---

## 7. Run John


---

## 3. The yescrypt Problem

I was testing John against a Linux password hash.

The hash started with:

```text
$y$...
```

This means the password was using **yescrypt**.

I checked whether my John build supported it:

```bash
john --list=formats | grep -i yescrypt
```

Nothing was returned.

When I tried to use the hash anyway:

```bash
john hashes.txt
```

I got:

```text
No password hashes loaded
```

So I needed a newer John build with yescrypt support.

---



## 5. Build a New Version

Instead of using the old APT package, I decided to build John from source.

First, I installed the required packages:

```bash
sudo apt install git build-essential libssl-dev zlib1g-dev
```

Then downloaded the source:

```bash
cd ~
git clone https://github.com/openwall/john.git
cd ~/john/src
```

Configured the build:

```bash
./configure
```

Then compiled it:

```bash
make -s clean
make -sj$(nproc)
```

After the build finished, the new John binary was located here:

```text
~/john/run/john
```

I used this binary instead of the old `john` command.

---

## 6. Use RockYou

Kali may have the RockYou wordlist compressed as:

```text
/usr/share/wordlists/rockyou.txt.gz
```

I checked:

```bash
ls -lah /usr/share/wordlists/
```

If necessary, I extracted it:

```bash
sudo gzip -d /usr/share/wordlists/rockyou.txt.gz
```

Now the wordlist was available as:

```text
/usr/share/wordlists/rockyou.txt
```

---

## 7. Run John Again

I ran the newly compiled version:

```bash
~/john/run/john --wordlist=/usr/share/wordlists/rockyou.txt ~/hashes.txt
```

This time John successfully loaded the hash:

```text
Loaded 1 password hash (crypt, generic crypt(3))
```

The important part was that the new build recognized the `yescrypt` hash.

The `--wordlist` option means this is a **dictionary attack**. John goes through the passwords in the wordlist and checks them against the hash.

If the password is not in the wordlist, John will not find it with this method.

---

## 8. Useful Commands

### Show the cracked password

```bash
~/john/run/john --show ~/hashes.txt
```

### Check the current status

```bash
~/john/run/john --status
```

### See supported formats

```bash
~/john/run/john --list=formats
```

### Search for crypt formats

```bash
~/john/run/john --list=formats | grep -i crypt
```

### Check how many CPU threads are available

```bash
nproc
```

### Remove John's saved results

John saves cracked hashes in:

```text
~/john/run/john.pot
```

For a fresh lab test, I can remove it:

```bash
rm ~/john/run/john.pot
```

Then recreate the hash file and run John again:

```bash
sudo unshadow /etc/passwd /etc/shadow > ~/hashes.txt

~/john/run/john --wordlist=/usr/share/wordlists/rockyou.txt ~/hashes.txt
```

---

## What I Learned

The main problem was not that John the Ripper was outdated as a project.

The problem was that the **Kali APT package was an older build** that did not support the `yescrypt` hash I was testing.

The solution was:

```text
Install from APT
       ↓
Check version
       ↓
No yescrypt support
       ↓
Build newer John from source
       ↓
Use ~/john/run/john
       ↓
yescrypt works
```

This was also a good example of why checking the actual version and supported formats is useful when a tool gives an error like `No password hashes loaded`.
