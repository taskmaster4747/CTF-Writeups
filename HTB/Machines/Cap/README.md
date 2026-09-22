Hack The Box – Cap Writeup





&#x20;Overview



• Machine Name: Cap

• Platform: Hack The Box

• Difficulty: Easy

• Objective: Gain User and Root access





&#x20;1. Reconnaissance



We start by scanning the target using Nmap:



nmap -sV 10.129.49.74



Findings:



• Port 21 → FTP (vsftpd)

• Port 22 → SSH

• Port 80 → HTTP (Web Application)



&#x20;Total 3 TCP ports open

&#x20;

2\. Web Enumeration



Accessing the web application on port 80 revealed a dashboard interface with a feature called Security Snapshot.



When triggering this feature, the browser redirects to:



/data/\[id]



&#x20;The \[something] = data



&#x20;3. IDOR Vulnerability (Broken Access Control)



By modifying the ID in the URL:



/data/1 → /data/2 → /data/0



We discovered:



• We could access other users' scan results

• This confirms an IDOR (Insecure Direct Object Reference) vulnerability



&#x20;Answer: Yes, we can access other users' scans

&#x20;

&#x20;4. PCAP File Discovery



Among the accessible scans, one contained a PCAP file with sensitive data.



• Identified the ID of the PCAP file containing credentials

&#x20;(Replace with actual ID you saw, e.g. 0 or 3 depending on your run)





&#x20;5. Traffic Analysis



The PCAP file was downloaded and analyzed using Wireshark.

Key Finding:



• Sensitive credentials were transmitted via FTP protocol

&#x20;Application Layer Protocol: FTP



6\. Credential Reuse From the PCAP:



• Found credentials for user nathan

Tried using same credentials on:

• FTP 

• SSH 



&#x20;Password reuse worked on SSH



&#x20;7. User Access



Logged in via SSH:



ssh nathan@10.129.49.74



Successfully accessed:



/home/nathan/user.txt



&#x20;User flag obtained



8\. Privilege Escalation



Checked for special permissions:



getcap -r / 2>/dev/null



Finding:



/usr/bin/python3 = cap\_setuid+ep

&#x20;This binary has special capabilities allowing privilege escalation



&#x20;Exploitation



Used Python to escalate privileges:



python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'



Now we have:



root@cap:/#



&#x20;9. Root Access



Accessed root flag:



/root/root.txt



&#x20;Root flag obtained







&#x20;Tools Used



• Nmap

• Browser (Manual testing)

• Wireshark

• SSH

• Python



&#x20;Key Learnings



• Learned how IDOR vulnerabilities expose sensitive data

• Understood how PCAP analysis can reveal credentials

• Practiced credential reuse attacks

• Learned Linux privilege escalation using capabilities (getcap)

• Gained experience in end-to-end exploitation workflow





&#x20;Conclusion



This machine demonstrates real-world attack paths:



• Weak access control (IDOR)

• Poor credential handling

• Misconfigured Linux capabilities



It reinforces the importance of:



• Proper authorization checks

• Secure credential transmission

• Least privilege principle





