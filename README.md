# John the Ripper — Kali Linux Lab

This guide explains how passwords are stored, what is **John the Ripper** and showing how I installed it on Kali, ran into a problem, and fixed it by building a newer version from source.

## Contents

- [0. How Does It Work](#0-how-does-it-work)
- [1. Install John](#1-install-john)
- [2. Prepare the Hash](#2-prepare-the-hash)
- [3. Running John](#3-running-john)
- [4. Hash Format Problem](#3-hash-format-problem)
- [5. Build a New Version](#5-build-a-new-version)
- [6. Use RockYou](#6-use-rockyou)
- [7. Run John Again](#7-run-john-again)
- [8. Useful Commands](#8-useful-commands)

---

## 0. How Does It Work

Passwords are usually not stored as plain text. Instead, they are stored as hashes, which are one-way representations of the original password. For example, a password SHA-256 hash might look like: `$y$j9T$5f7K3mQx8Wz2LrNp$QvX9mN8kR3sT6pY2aBcD4eF7gH1jK5lM9nP2qR6sT8uV`

**_John the Ripper_** is a open-source password-cracking tool that allows us to test password strength. Its goal is simple: take a list of password hashes (dictionary) and find the original plaintext passwords by hashing guesses and and compares it with the target hash. If they match, John has found the original password.

Since John works directly with password hashes, it does not need to interact with the original system. It can test a large number of password guesses without causing login attempts, lockouts, or rate limits. In an offline attack, cracking speed mainly depends on the available hardware and how much time you have.

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

## 3. Running John

At this point, the basic process is simple: run John and give it the text file containing the hashes:

```bash
john hashes.txt
```

John should then start going through password dictionary and comparing them with the hash.

This is basically what I expected from the password-cracking process covered in my **CompTIA A+** course. But in my case, it didn't work. **John** returned:

```text
No password hashes loaded
```
...even thought they were loaded into `hashes.txt` file. At this point I started digging deeper into what was actually happening.

---

## 4. Hash Format Problem

There are many common hash formats, for example:

```text
$1$...       → MD5
$5$...       → SHA-256
$6$...       → SHA-512
$2b$...      → bcrypt
$y$...       → yescrypt
```

My problem was the `$y$` hash format, **yescrypt**, which the old version of **John the Ripper (JtR)** could not read. The **JtR** version was **1.9.0 Jumbo**. The Kali repository was providing an older stable package, while newer **JtR** development had continued upstream. To get a newer version with the required support, you need to download the source from **[developer's GitHub](https://github.com/openwall/john)** and compile it yourself.

---


## 5. Build a New Version

First, install the required packages:

```bash
sudo apt install git build-essential libssl-dev zlib1g-dev
```

Then downloaded the source into home directory:

```bash
cd ~
git clone https://github.com/openwall/john.git
cd ~/john/src
```

Configured the build:

```bash
./configure
```

Then we see that configure finished and to compiled it we have to run:

```bash
make -s clean && make -sj6
```

After the build finished, the new John binary was located here:

```text
~/john/run/john
```

I used this binary instead of the old `john` command.

---

## 6. Use RockYou

My Kali didn't have the RockYou wordlist. So I installed it:

```text
sudo apt update
sudo apt install wordlists
```
After installation it can be found there:
```text
/usr/share/wordlists/rockyou.txt.gz
```

I checked:

```bash
ls -lah /usr/share/wordlists/
```

Extract it:

```bash
sudo gzip -d /usr/share/wordlists/rockyou.txt.gz
```

Now the wordlist was available as:

```text
/usr/share/wordlists/rockyou.txt
```

---

## 7. Run John Again

Run the newly compiled version. Specify the path to the wordlist it will use for comparison:

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

John keeps a `john.pot` file containing all cracked passwords.

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

### John's saved results

John saves cracked hashes in:

```text
~/john/run/john.pot
```

For each new password-cracking test, you need to clear John's saved results:

```bash
rm ~/john/run/john.pot
rm ~/hashes.txt
```

...and recreate the hash file:

```bash
sudo unshadow /etc/passwd /etc/shadow > ~/hashes.txt

```
