# AWS EC2 File Transfer Guide (SCP)

A complete step-by-step guide for transferring files and folders between an **AWS EC2 Instance** and your **Local Machine (Windows, macOS, or Linux)** using **SCP (Secure Copy Protocol)**.

---

# What is SCP?

**SCP (Secure Copy Protocol)** is a command-line tool used to securely transfer files and directories between:

- Local Computer → EC2 Instance
- EC2 Instance → Local Computer
- One Remote Server → Another Remote Server

SCP uses **SSH** for authentication and encryption, making file transfers secure.

---

# Prerequisites

Before using SCP, make sure you have:

- Your EC2 instance is running.
- The EC2 Public IP or Public DNS Name.
- The `.pem` key used while launching the instance.
- SSH access working properly.
- A terminal (Linux/macOS) or Command Prompt / PowerShell / Git Bash (Windows).

---

# Important Rule

> **Always run SCP commands from your LOCAL computer.**

Do **NOT** execute SCP commands inside an SSH session connected to the EC2 instance.

Correct:

```
Your Local PC Terminal
        │
        ├── ssh
        └── scp
```

Wrong:

```
Local PC
    │
    └── SSH into EC2
            │
            └── Run scp ❌
```

---

# Default EC2 Usernames

| Operating System | Username |
|------------------|----------|
| Amazon Linux | ec2-user |
| Amazon Linux 2 | ec2-user |
| Ubuntu | ubuntu |
| Debian | admin |
| CentOS | centos |
| RHEL | ec2-user |

---

# Basic SCP Syntax

```bash
scp [OPTIONS] -i "PATH_TO_KEY.pem" USER@HOST:REMOTE_PATH LOCAL_PATH
```

### Breakdown

```
scp
```

The Secure Copy command.

```
-i
```

Specifies the private key.

```
PATH_TO_KEY.pem
```

Location of your downloaded AWS key.

Example:

```text
C:\Users\Krishna\Downloads\mykey.pem
```

or

```text
~/Downloads/mykey.pem
```

```
USER
```

EC2 username.

Example:

```
ec2-user
```

or

```
ubuntu
```

```
HOST
```

EC2 Public IP or Public DNS.

Example:

```
13.222.190.141
```

or

```
ec2-13-222-190-141.compute-1.amazonaws.com
```

```
REMOTE_PATH
```

Location of the file on EC2.

Example:

```
/home/ec2-user/file.txt
```

```
LOCAL_PATH
```

Location on your computer.

Example:

```
Downloads/
```

---

# File Transfer Directions

There are only two possibilities.

## 1. Local → EC2

Upload a file from your computer to the EC2 instance.

```
Local PC
    │
    ▼
   EC2
```

Example:

```bash
scp -i "mykey.pem" notes.txt ec2-user@13.222.190.141:/home/ec2-user/
```

Meaning:

- Upload `notes.txt`
- Authenticate using `mykey.pem`
- Connect as `ec2-user`
- Store the file inside `/home/ec2-user/`

---

## 2. EC2 → Local

Download a file from EC2 to your computer.

```
EC2
 │
 ▼
Local PC
```

Example:

```bash
scp -i "mykey.pem" ec2-user@13.222.190.141:/home/ec2-user/report.pdf Downloads/
```

Meaning:

- Download `report.pdf`
- Save it inside the local `Downloads` folder

---

# Copying Directories

To copy an entire folder, use the `-r` (recursive) option.

## Local Folder → EC2

```bash
scp -r -i "mykey.pem" project ec2-user@13.222.190.141:/home/ec2-user/
```

---

## EC2 Folder → Local

```bash
scp -r -i "mykey.pem" ec2-user@13.222.190.141:/home/ec2-user/project Downloads/
```

---

# Common SCP Options

| Option | Purpose |
|---------|---------|
| `-i` | Private key |
| `-r` | Copy directories recursively |
| `-P` | SSH port (uppercase P) |
| `-v` | Verbose mode for debugging |
| `-C` | Compress data during transfer |
| `-q` | Quiet mode |

Example:

```bash
scp -v -i "mykey.pem" file.txt ec2-user@13.222.190.141:/home/ec2-user/
```

---

# File Path Examples

## Linux (EC2)

```text
/home/ec2-user/file.txt
```

```text
/var/www/html/index.html
```

```text
/etc/nginx/nginx.conf
```

---

## Windows

```text
C:\Users\Krishna\Downloads\
```

---

## macOS/Linux

```text
~/Downloads/
```

---

# Example Commands

## Upload a file

```bash
scp -i "mykey.pem" image.png ec2-user@13.222.190.141:/home/ec2-user/
```

---

## Download a file

```bash
scp -i "mykey.pem" ec2-user@13.222.190.141:/home/ec2-user/image.png .
```

---

## Upload an entire project

```bash
scp -r -i "mykey.pem" myproject ec2-user@13.222.190.141:/home/ec2-user/
```

---

## Download an entire directory

```bash
scp -r -i "mykey.pem" ec2-user@13.222.190.141:/var/www/html ./website
```

---

# Common Errors and Solutions

## Permission denied (publickey)

Cause:

- Wrong username
- Wrong `.pem` file
- Incorrect key permissions

Solution:

- Verify the correct username.
- Use the correct `.pem` file.
- Ensure the key has proper permissions (`chmod 400 mykey.pem` on Linux/macOS).

---

## No such file or directory

Cause:

Incorrect file path.

Solution:

Check the file location using:

```bash
ls
```

or

```bash
pwd
```

---

## Connection timed out

Cause:

- EC2 instance stopped
- Security Group blocks port 22
- Incorrect Public IP

Solution:

- Start the EC2 instance.
- Allow inbound SSH (TCP port 22) in the Security Group.
- Verify the correct Public IP or DNS.

---

## Permission denied

Cause:

Trying to copy into a protected directory.

Solution:

Copy the file into your home directory first, then move it with `sudo` if needed.

---

# Quick Cheatsheet

## Upload File

```bash
scp -i "key.pem" file.txt ec2-user@HOST:/home/ec2-user/
```

---

## Download File

```bash
scp -i "key.pem" ec2-user@HOST:/home/ec2-user/file.txt .
```

---

## Upload Folder

```bash
scp -r -i "key.pem" folder ec2-user@HOST:/home/ec2-user/
```

---

## Download Folder

```bash
scp -r -i "key.pem" ec2-user@HOST:/home/ec2-user/folder .
```

---

# Remember

- SCP uses SSH.
- Run SCP commands only from your local machine.
- Use `-i` to specify your AWS private key.
- Use `-r` to copy folders.
- The remote path is written after `USER@HOST:`.
- Verify the username based on your EC2 operating system.
