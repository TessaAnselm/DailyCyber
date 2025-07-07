**Skilltest: Information Gathering**

This is HTB skillset on information gathering walkthrough that goes over reconnaissance of inlanefreight.htb using VHost enumeration

The domain inlanefreight.htb is a virtual host that doesn’t exist in public DNS.  
The first step is to add the target IP to your /etc/hosts file for proper domain resolution.

Added target ip into /etc/hosts  
![2832857cb4c1d63c943dc805abc7af3c.png](resources/2832857cb4c1d63c943dc805abc7af3c.png)

You can confirm connectivity with:  
`curl -I http://inlanefreight.htb:42213`

To find the software agent, you can use whatweb:  
![4105353c8178a04b6c77b3e7c86ea2a9.png](resources/4105353c8178a04b6c77b3e7c86ea2a9.png)

* * *

**Enumerate Hidden Virtual Hosts**

You can use ffuf or gobuster to enumerate vHosts (virtual hosts).  
Command used for fuff:

```
ffuf -u http://94.237.61.242:42213/ \
     -H "Host: FUZZ.inlanefreight.htb" \
     -w SecLists/Discovery/DNS/subdomains-top1million-110000.txt \
     -fs 120 -fc 404 \
     -t 200
```

Command used for gobuster

```
gobuster vhost -u http://inlanefreight.htb:42213 -w SecLists/Discovery/DNS/subdomains-top1million-110000.txt --append-domain -t 200

```

Result found web1337:  
![016968995292cdf1b1d4859d8e1dc96f.png](resources/016968995292cdf1b1d4859d8e1dc96f.png)

Added discovered vHost entry to /etc/hosts for domain resolution:  
![868e99be4019c13e3d47dea30b805061.png](resources/868e99be4019c13e3d47dea30b805061.png)

URL of interest found in robots.txt  
![7ae8ad06c0b4e7ddf83e21ccbcdc9f66.png](resources/7ae8ad06c0b4e7ddf83e21ccbcdc9f66.png)

API key found by looking into the disallow robots.txt  
![4f2d8c306d75b0e87a1c3f02ef13b28a.png](resources/4f2d8c306d75b0e87a1c3f02ef13b28a.png)

You can also run command  
`curl -s http://web1337.inlanefreight.htb:42213/admin_h1dd3n/`

* * *

**Setup Scrapy and ReconSpider in Virtual Environment**

Used virtual environment to add scrapy and ReconSpider

```
pip3 install scrapy
wget -O ReconSpider.zip https://academy.hackthebox.com/storage/modules/144/ReconSpider.v1.2.zip
unzip ReconSpider.zip 
```

Run the following

```
python3 ReconSpider.py http://{SUBDOMAIN}.inlanefreight.htb:{PORT}
 
```

EX:  
`python3 ReconSpider.py http://web1337.inlanefreight.htb:42213`  
No results were returned initially.

To capture output and errors: (must include standard error input to get the error info)

```
python3 ReconSpider.py http://web1337.inlanefreight.htb:42213 > ~/Desktop/inlane_lists 2>&1

```

* * *

**Re-Enumerate Above the Subdomain**

Since web1337.inlanefreight.htb gave no useful output, I enumerated vHosts above it using gobuster:

```
gobuster vhost -u http://web1337.inlanefreight.htb:40656   -w SecLists/Discovery/DNS/subdomains-top1million-110000.txt   --append-domain -t 200

```

Alternatively with ffuf:

```
ffuf -u http://94.237.122.145:40656/ \
     -H "Host: FUZZ.web1337.inlanefreight.htb" \
     -w SecLists/Discovery/DNS/subdomains-top1million-110000.txt \
     -fs 120 -fc 404 \
     -t 200
```

* * *

**Results from New Subdomain**

Once a new vHost was discovered, crawling it with ReconSpider revealed email information and another disclose API key as follows:

Email found:  
![107f0c3476a752b3c49bbbb29f72aa34.png](resources/107f0c3476a752b3c49bbbb29f72aa34.png)

API key found under comments:  
![5ef5cad612211d758d19323be026db25.png](resources/5ef5cad612211d758d19323be026db25.png)
