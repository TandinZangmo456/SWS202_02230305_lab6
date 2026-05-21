**Lab 06: Data Exfiltration & Persistent Access**

Cybersecurity Lab Report

Student: Tandinzam  |  Date: May 19, 2026  |  Environment: Kali Linux vs Metasploitable2

# **Part A — Initial Access**

## **Task 1: Network Discovery**

Nmap was used to scan the target VM (192.168.56.5) for open services. Four services were discovered: vsftpd 2.3.4 on port 21, OpenSSH 4.7p1 on port 22, Apache 2.2.8 on port 80, and Samba on port 445\. The vsftpd 2.3.4 service was identified as vulnerable due to a known backdoor (CVE-2011-2523). Version detection is critical during reconnaissance as it maps exact software versions to known CVEs, enabling targeted exploitation.

![alt text](<SS/Screenshot from 2026-05-19 22-25-30.png>)

*Figure 1: Nmap scan results showing discovered services and versions on target 192.168.56.5*

## **Task 2: Exploiting the Vulnerability**

Metasploit was launched and the vsftpd\_234\_backdoor exploit module was loaded. After setting RHOSTS to 192.168.56.5 and LHOST to 192.168.56.4, the exploit was executed successfully, opening Meterpreter session 1\. Root access (uid=0) was confirmed immediately. Defenders can detect this by monitoring IDS signatures for vsftpd exploit patterns and alerting on unexpected outbound connections on non-standard ports.

![alt text](<SS/Screenshot from 2026-05-19 22-26-37.png>)

*Figure 2: Metasploit console launched and vsftpd module searched*

![alt text](<SS/Screenshot from 2026-05-19 22-26-58.png>)

*Figure 3: Exploit module configured with RHOSTS and options shown*

![alt text](<SS/Screenshot from 2026-05-19 22-27-26.png>)

*Figure 4: Exploit executed — Meterpreter session opened, root shell confirmed (uid=0)*

# **Part B — Data Exfiltration**

## **Task 3: Identifying Sensitive Files**

Using the find command, numerous sensitive files were located on the target system. Configuration files (.conf) included ALSA and AppArmor settings, while database files (.db) included Firefox certificate stores and mail aliases. The /etc/passwd file revealed 35 user accounts including service accounts for MySQL, PostgreSQL, Tomcat, and human users such as msfadmin and user. Configuration files are valuable to attackers because they often contain credentials, network settings, and service configurations.

![alt text](<SS/Screenshot from 2026-05-19 22-28-06.png>)

*Figure 5: Sensitive .conf and .db files discovered on target system*

![alt text](<SS/Screenshot from 2026-05-19 22-28-30.png>)

*Figure 6: Contents of /etc/passwd showing all user accounts*

## **Task 4: Exfiltration via Netcat**

A Netcat listener was established on the attacker machine (port 4444\) to receive data. From the target, /etc/passwd was piped through Netcat to the attacker. The transfer was successful — received\_data.txt was 1.6KB with 36 lines, exactly matching the source file. Defenders should monitor for unusual raw TCP connections, especially on non-standard ports, and alert on Netcat usage within the network.

![alt text](<SS/Screenshot from 2026-05-19 22-29-30.png>)

*Figure 7: Netcat listener received /etc/passwd — 1.6KB, 36 lines confirmed*

## **Task 5: Exfiltration via HTTP POST**

A Python HTTP server was started on port 8080 on the attacker machine. A curl POST request was sent from the target containing the /etc/passwd file. The server logged the incoming POST request confirming successful data transfer. HTTP-based exfiltration is harder to detect because it blends with legitimate web traffic. Suspicious indicators include large POST requests to non-web servers, unexpected destinations, and unusual user-agent strings.

![alt text](<SS/Screenshot from 2026-05-19 22-30-01.png>)

*Figure 8: HTTP POST exfiltration via curl — server returned 501 confirming connection reached attacker*

# **Part C — Persistence Mechanisms**

## **Task 6: Creating Persistence**

Three persistence methods were deployed. Method 1: A cron job was added to /var/spool/cron/root to execute a reverse shell every 5 minutes. Method 2: An RSA public key was injected into /root/.ssh/authorized\_keys, allowing keyless SSH login. Method 3: A backdoor startup script was created at /etc/init.d/update-service. SSH key injection is the hardest to detect as it appears as legitimate authentication. Defenders can audit cron jobs via crontab \-l and inspect authorized\_keys files regularly.

![alt text](<SS/Screenshot from 2026-05-19 22-31-18.png>)

*Figure 9: Cron job added and SSH key pair generated on target*

![alt text](<SS/Screenshot from 2026-05-19 22-33-51.png>)

*Figure 10: SSH key injected into authorized\_keys, backdoor service created, netstat shows active connection*

# **Part D — Detection and Cleanup**

## **Task 7: Detecting Exfiltration and Persistence**

Using netstat, the active reverse shell connection (192.168.56.5:38944 to 192.168.56.4:4444 ESTABLISHED) was identified. Inspecting /var/spool/cron/root revealed the malicious cron job, and reviewing authorized\_keys exposed the injected SSH key. Tools like netstat, tcpdump, crontab audits, and log analysis are essential for incident response and should be part of regular security monitoring.

## **Task 8: Cleanup and Remediation**

All persistence mechanisms were successfully removed: the cron job was deleted with crontab \-r and rm, the injected SSH key was removed using sed, and the backdoor service script was deleted. The Meterpreter session closed automatically confirming full remediation. Remediation is critical to prevent re-entry and lateral movement. Additional hardening includes patching vsftpd, disabling unused services, implementing host-based IDS, and conducting regular SSH key audits.

![alt text](<SS/Screenshot from 2026-05-19 22-34-18.png>)

*Figure 11: Detection and cleanup — cron removed, SSH key cleaned, service deleted, Meterpreter session closed*

# **Conclusion**

This lab demonstrated the complete attack lifecycle from reconnaissance to remediation. Using Nmap, the vulnerable vsftpd 2.3.4 service was identified and exploited via Metasploit to obtain root access. Sensitive files were located and exfiltrated using both Netcat and HTTP POST. Three persistence mechanisms were established: a cron-based reverse shell, SSH key injection, and a backdoor init script. The detection phase confirmed all mechanisms were identifiable through standard defensive tools including netstat, crontab audits, and SSH key reviews. Successful remediation involved removing all persistence artifacts. Key lessons include the importance of patch management, outbound traffic monitoring, and regular authentication audits as core defensive practices against real-world attacks.
