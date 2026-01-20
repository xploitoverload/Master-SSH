---
SSH 
---

connecting ssh

==========================

ssh ip
enter user
enter pass(if available)
enter mfa(if available)

---------------------------

ssh user@ip
enter pass(if)

ssh user:pass@ip

---------------------------
===========================
using hosts  
===========================
'me clear that hosts and hostname are different in their own way, so if u dont know google it or man ssh

edit /etc/hosts

many ways to edit it (make sure u dont overwrite it in either way)

so here is the best practices for it
------------------------------------------------
echo "<ip> <host>" | sudo tee -a /etc/hosts
------------------------------------------------
or
------------------------------------------------
sudo echo "<ip> <host>" >> /etc/hosts
------------------------------------------------
or
------------------------------------------------
use ur fav editor such as vi/vim, nano or mousepad  
------------------------------------------------

Now,,
-------------------------------------------------
ssh host   ---> it will take default use wont asks for user

ssh user@host 
enter pass (if)

ssh user:pass@host
--------------------------------------------------

==================================================
Edit SSH Config
==================================================
location : ~/.ssh/config

when any error comes regarding host-key algorithms
--------------------------------------------------
Temp fix:
ssh -o HostKeyAlgorithms=+rsa-sha2-512,rsa-sha2-256 user@ip
--------------------------------------------------
you can do this but this will affect all ssh connections, so not suggesting this one 
echo "HostKeyAlgorithms +rsa-sha2-512,rsa-sha2-256" >> ~/.ssh/config
---------------------------------------------------
Permanent fix:

Add this to ~/.ssh/config

Host <name-or-hostname>
    HostName <ip>
    User root
    HostKeyAlgorithms +rsa-sha2-512,rsa-sha2-256
    PubkeyAcceptedAlgorithms +rsa-sha2-512,rsa-sha2-256
    
now,

ssh name-or-hostname
---------------------------------------------------
Generate modern host-key: This will fix long-term

ssh-keygen -A

systemctl restart sshd
or
/etc/init.d/sshd restart

ssh <user>@<ip>
----------------------------------------------------
when any err comes like Connection timeout / no route to host

ping <ip>
ip route
ifconfig -a

ip link set eth0 up
udhcpc -i eth0

------------------------------------------------------
Connection refused:
ps | grep sshd
/etc/init.d/sshd start
or
systemctl start sshd
-------------------------------------------------------

when u see something like this WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!

ssh-keygen -R <ip>

--------------------------------------------------------
if u find no matching key exchange method found

temp:
-o KexAlgorithms=+diffie-hellman-group14-sha1

perm:
add this to ~/.ssh/config

KexAlgorithms +diffie-hellman-group14-sha1

-----------------------------------------------------------

There are so many errors and common fix for those erros
such like slow logins, ssh drops randomly, firewall blocks ssh, dropbear vs ssh mismatch and so on

So if u need it in emergency do it quickly
-----------------------------------------------------------
Host <name>
  HostName <ip>
  User root
  HostKeyAlgorithms +rsa-sha2-512,rsa-sha2-256
  PubkeyAcceptedAlgorithms +rsa-sha2-512,rsa-sha2-256
  ServerAliveInterval 30
-----------------------------------------------------------


