Author: Ray1N
Client: TryHackMe Room(Internal)
Objective: Assecc the security posture of the target host and determine whether administrative access can be obtained.
Date: June 2025


NMAP Scan: [ nmap -A -sS -sC -sV -O -v ***** ]
A comprehensive TCP post scan identified two exposed services: SSH(22/TCP) and HTTP(80/TCP).The presence of a web service indicated a potential attack surface and was selected for further enumeration.
![[TCP.png]]

----------------------------

Directory Brute-forcing
I’ll use Gobuster to discover hidden directories: [ gobuster dir -u ******** -w /usr/share/wordlists/dirb/common.txt ]

/blog - 301
/wordpress - 301
/phpmyadmin - 301
/javascript - 301

----------------------------

WordPress Enumeration: [ wpscan --url ******/blog -e u ]
WordPress Version: 5.4.2 (Latest at the time)
Theme: Twenty Seventeen v2.3 (outdated)
XML-RPC: Enabled (potential attack vector)
User Identified: admin
The admin user is a common WordPress username, making it a prime target for password attacks.

----------------------------
WPScan — Password Brute Force: [ wpscan --url ******/blog -U admin -P /usr/share/wordlists/*** ]

Username: admin
Password: ************
![[screenshot_260604_123526.png]]

----------------------------
Authenticaded Code Execution via Theme Editor

During the assessment, valid WordPress administrative credentials were obtained, providing access to the administrative dashboard.
Review of the available functionality revealed access to the built-in Theme Editor feature. This component allows administrators to directly modify PHP template files from the web interface.
The custom 404 error page template was selected as an execution point because it is publicly accessible and can be triggered on demand by requesting a non-existent resource.
A PHP reverse shell payload was inserted into the template and saved through the administrative interface. Once the modified page was requested, the payload executed on the server and established an outbound connection to the attacker's listener.
As a result, remote code execution was achieved under the context of the web server process.
Impact:
Arbitrary command execution on the target host.
Ability to interact with the underlying operating system.
Establishment of an initial foothold for further post-exploitation activities.

A PHP reverse shell payload was embedded into the 404 template to archive remote code execution
---------------------------

Post-Exploitation Enumeration
Following successful remote code execution, an interactive shell was obtained under the context of the web server account.
Plain text
www-data
At this stage, the objective shifted from initial access to privilege escalation. A systematic enumeration of the host was performed to identify misconfigurations, sensitive files, stored credentials, and privileged services that could facilitate further access.
Key areas of investigation included:
User home directories
Application configuration files
Scheduled tasks (cron jobs)
SUID/SGID binaries
Running services
Stored credentials and secrets
Internal-only applications
SSH keys and authentication material
Particular attention was given to files commonly used by web applications, as these frequently contain database credentials, API keys, or reused passwords that may enable lateral![[screenshot_260604_124551.png]] movement or privilege escalation.

The enumeration process ultimately focused on identifying valuable credentials and internal services accessible from the compromised host.
----------------------------
Credential Discovery
During post-exploitation enumeration, a review of files located outside the web application directory identified a text file stored within the /opt directory.
The file contained an internal message referencing operational credentials intended for another user account. The content suggested that sensitive authentication data had been stored in plaintext and was accessible to the compromised web server account.
This finding demonstrated inadequate credential management practices and exposed reusable authentication material to any attacker who successfully compromised the web application.
Impact:
Exposure of valid user credentials.
Potential for privilege escalation through credential reuse.
Increased risk of lateral movement within the environment.
User Account Compromise
The recovered credentials were validated against local system accounts and were found to be associated with a legitimate user.
Authentication using the discovered credentials was successful, allowing a transition from the low-privileged web server context to a standard user account.

![[screenshot_260604_125616.png]]This represented a significant increase in access and enabled further enumeration of the host from a more privileged position within the operating system.
-----------------------------
Privilege Escalation Through Credential Reuse
Following the discovery of exposed credentials, successful authentication was performed against a valid local user account. This resulted in a transition from the restricted web server context to a legitimate operating system user.
The ability to move from a service account to an authenticated user represents a critical escalation in attacker capabilities. Unlike the initial web server shell, a user account provides greater access to system resources, personal files, application data, and internal services.
At this stage, the compromise extended beyond the web application itself and resulted in direct access to the underlying operating system.
Security Impact:
Compromise of a legitimate user account.
Access to sensitive user-owned files and directories.
Ability to harvest additional credentials and secrets.
Increased visibility into internal services and configurations.
Establishment of a stable foothold for further privilege escalation activities.
The successful transition from a web service account to a valid system user demonstrated the effectiveness of credential reuse as a post-exploitation technique and significantly increased the overall risk associated with the compromise.
----------------------------------
Internal Service Discovery
Following successful compromise of a local user account, additional host enumeration was conducted to identify internal services that were not directly exposed to external users.
Network inspection techniques were employed to identify listening services and active interfaces within the host environment.
Bash
ip addr
ss -tulpn
During this process, an additional network interface associated with a Docker environment was identified.
Plain text
docker0
172.17.0.1/16
Further investigation revealed an internal service listening on the Docker network:
Plain text
172.17.0.2:8080
The service was not accessible externally and appeared to be intended for internal use only. However, because the attacker had already obtained local access to the host, these restrictions no longer provided meaningful protection.
The discovery of the service significantly expanded the available attack surface and presented a potential avenue for further privilege escalation.
Security Impact:
Exposure of internal-only services to compromised users.
Increased attack surface following host compromise.
Potential access to administrative interfaces and sensitive data.
![[screenshot_260604_130219.png]]
![[screenshot_260604_130056.png]]
Additional opportunities for privilege escalation.
-----------------------------------
Accessing Internal Services via SSH Port Forwarding
During host enumeration, an internal web service was identified within the Docker network and was bound exclusively to an internal IP address.
Because the service was not directly reachable from the external network, a secure SSH tunnel was established through the compromised user account to access the application remotely.
Bash
ssh -L 8080:172.17.0.2:8080 aubreanna@***********
This configuration forwarded local TCP port 8080 on the assessment workstation to the internal Docker service listening on 172.17.0.2:8080.
As a result, the previously inaccessible application became available through the local browser interface:
Plain text
http://127.0.0.1:8080
The service was identified as a Jenkins automation server.
The discovery was significant because Jenkins frequently operates with elevated privileges and often contains sensitive information, including build configurations, stored credentials, API tokens, and administrative functionality.
The successful use of SSH port forwarding effectively bypassed network segmentation controls and provided direct access to an internal service that was never intended to be exposed externally.
Security Impact:
Exposure of internal administrative services following host compromise.
Circumvention of network isolation controls.
Expansion of the attack surface beyond externally accessible services.
![[screenshot_260604_131020.png]]
Potential access to privileged administrative functionality.
---------------------------------------
Authentication Assessment
Following access to the internal Jenkins instance, the authentication mechanism was evaluated to determine the resilience of the platform against password-based attacks.
A controlled password auditing exercise was conducted against identified user accounts using a predefined credential list. The objective was to assess whether weak or commonly used passwords could be successfully leveraged to gain access to the administrative interface.
The assessment revealed that one of the accounts was protected by an insufficiently complex password, allowing successful authentication.
This finding demonstrated that the security of the Jenkins instance relied heavily on password strength and lacked additional safeguards that could mitigate credential-based attacks.
Impact
Successful authentication to the Jenkins platform granted access to administrative functionality and sensitive configuration data. Depending on the permissions assigned to the compromised account, an attacker could potentially:
Access build configurations.
Review stored credentials and secrets.
Execute build jobs.
Interact with underlying host resources.
Facilitate further privilege escalation activities.
Risk
Severity: High
The use of weak credentials significantly increases the likelihood of unauthorized access, particularly when administrative services are reachable from compromised internal hosts.
Recommendations
Enforce a strong password policy.
Implement multi-factor authentication.
Restrict authentication attempts through rate limiting and account lockout mechanisms.
Periodically audit user credentials for weak passwords.
![[screenshot_260604_132122.png]]

![[screenshot_260604_132423.png]]
Monitor authentication logs for suspicious activity.
-----------------------------------------------
Jenkins Secret Exposure
During the assessment of the Jenkins instance, access to administrative functionality enabled the review of stored configuration data and sensitive operational information.
Further analysis revealed credentials and secrets associated with privileged services. The exposed information provided a viable path for further escalation and ultimately facilitated administrative access to the underlying environment.
Impact:
Exposure of privileged credentials.
Compromise of infrastructure management services.
Potential full-system compromise.

Administrative Access Achieved
The assessment concluded with the successful acquisition of administrative access to the target system.
Through a combination of web application compromise, credential exposure, insufficient network isolation, and weaknesses in internal service security, it was possible to progress from an unauthenticated external position to full system compromise.
Administrative access provided unrestricted control over the host, including the ability to access sensitive data, modify system configurations, manage user accounts, and execute arbitrary commands with the highest level of privileges.
This demonstrated that a determined attacker could achieve complete compromise of the environment by exploiting a chain of individually preventable security weaknesses.
Conclusion
The assessment identified multiple security issues that, when combined, enabled full compromise of the target system.
While no single vulnerability alone guaranteed administrative access, the cumulative effect of the findings created a clear attack path from initial reconnaissance to complete host takeover.
The engagement highlighted the importance of defense in depth. Weaknesses in credential management, application security, and internal service exposure significantly reduced the effort required to escalate privileges and maintain access to the environment.
The successful compromise of the system should be considered a critical security event, as an attacker with administrative privileges would have unrestricted access to all resources hosted on the affected server.
Recommendations
The following remediation measures are recommended to reduce the likelihood of similar compromises in the future:
Web Application Security
Restrict access to administrative interfaces.
Regularly update WordPress core, plugins, and themes.
Disable direct code editing through the WordPress administration panel.
Conduct periodic vulnerability assessments of public-facing applications.
Credential Management
Eliminate plaintext credential storage.
Implement centralized secret management solutions.
Enforce strong password complexity requirements.
Rotate privileged credentials regularly.
Internal Service Protection
Limit access to administrative services such as Jenkins.
Implement role-based access controls.
Enable multi-factor authentication for administrative accounts.
Review and remove unnecessary privileged accounts.
Monitoring and Detection
Enable centralized logging and security monitoring.
Monitor authentication failures and suspicious administrative activity.
Establish alerting for privilege escalation events.
Conduct regular security reviews of internal infrastructure.
System Hardening
Apply the principle of least privilege across all services and user accounts.
Remove unnecessary software and services.
Regularly patch operating systems and applications.
Perform periodic configuration audits to identify security misconfigurations.
Final Assessment
Overall Risk Rating: Critical
The assessment demonstrated that an attacker could successfully transition from an external unauthenticated position to full administrative control of the host. Immediate remediation of the identified findings is strongly recommended to reduce organizational risk and improve the overall security posture of the environment
