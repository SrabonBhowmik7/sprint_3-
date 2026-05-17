Sprint 3 — Phishing & Malware simulation

MITRE ATT&CK: T1566.001 — Spearphishing Attachment 

**Prerequisites — confirm before starting**
Activity 3 (Email Server) fully complete
Postfix and Dovecot running on UbuntuServer (192.168.1.80)
Thunderbird configured on UbuntuDesktop with desktop-user account
Two users created: server-user and desktop-user (password: password)
UbuntuDesktop can send and receive emails via Thunderbird
This is the focused Sprint 3 manual. Use the tabs to navigate between:
●	Prerequisites — confirm everything is ready before starting
●	Setup — install swaks and create the fake malware file
●	Attack steps — 7 numbered steps with exact commands and expected outputs and screenshots for this attacks.
●	Defense — SPF, DKIM, ClamAV and other mitigations to document


Setup — prepare your attack tools 
Step 1 — Install swaks on UbuntuServer
swaks (Swiss Army Knife for SMTP) lets you craft and send fully customised emails from the command line, including spoofed From addresses and attachments.
sudo apt install swaks -y
Verify with:
swaks --version
Run on: UbuntuServer (192.168.1.80)
<img width="975" height="908" alt="image" src="https://github.com/user-attachments/assets/66042be9-6c65-45fe-a930-c89e5e0294a5" />



Step 2 — Verify Postfix is running
Confirm the mail server is active before sending any test emails.
sudo systemctl status postfix
sudo ss -tuln | grep :25
Port 25 must be listed as LISTEN. If not, run: sudo systemctl restart postfix

 

Step 3 — Create a harmless simulated malware file
This script does nothing harmful — it just prints a ransomware-style message to simulate what a malware payload looks like when delivered via email.
echo '#!/bin/bash
echo "System compromised."
echo "All files encrypted. Pay 1 BTC to recover your data."
' > ~/malware_simulation.sh
chmod +x ~/malware_simulation.sh
cat ~/malware_simulation.sh
Run on: UbuntuServer. This file will be attached to the phishing email in Step 6.
 

Attack steps and evidence — execute in order 


1.	Send a basic phishing email (no attachment)
On UbuntuServer, use swaks to send a spoofed urgent email to desktop-user. The From address is faked to appear as the IT security team.
swaks --to desktop-user@hasibulwilproject.com \
      --from security@hasibulwilproject.com \
      --server 192.168.1.80 \
      --port 25 \
      --header “ Subject: URGENT: Your account will be suspended" \
      --body "Dear User,

Your account has been flagged for suspicious activity.
Click here immediately to verify your identity:
http://fake-login.hasibulwilproject.com

Failure to act within 24 hours will result in permanent suspension.

IT Security Team"
 

2.	Verify delivery in mail logs
Check the Postfix mail log to confirm the email was sent successfully.
sudo grep "status=sent" /var/log/syslog 

 

3.	Confirm phishing email arrived in Thunderbird
On UbuntuDesktop, open Thunderbird and check the inbox of desktop-user@hasibulwilproject.com. The spoofed email from security@hasibulwilproject.com should be visible.
 

4.	Send a second phishing email with a malware attachment
Now attach the simulated malware script to a second phishing email, disguised as a legitimate system update.
swaks --to desktop-user@hasibulwilproject.com \
      --from it-support@hasibulwilproject.com \
      --server 192.168.1.80 \
      --port 25 \
      --header “Subject: Action required: Run security patch now" \
      --body "Dear User,

A critical security vulnerability has been identified on your system.
Please run the attached patch file immediately to protect your account.

This is mandatory. Failure to comply may result in data loss.

IT Support Team" \
      --attach ~/malware_simulation.sh
 
 




5. Verify the attachment arrived in Thunderbird
On UbuntuDesktop, open Thunderbird. The second email should appear with the .sh file attached. Open the email and confirm the attachment is visible.
 


6. Check email headers to analyse spoofing
In Thunderbird, right-click the phishing email and select View Source or More > View Source. Look at the Received and From headers to understand how spoofing works.

 

7. Save mail log evidence for your report
Extract relevant log entries and save them as a text file for your attack report.
sudo grep "desktop-user" /var/log/syslog > ~/phishing_evidence.txt
cat ~/phishing_evidence.txt


 


Mitigation & defense strategies 

SPF — Sender Policy Framework
SPF records specify which mail servers are authorised to send email for your domain. This prevents attackers from spoofing your From address.
# Add to your DNS zone file on InternalGateway:
@ IN TXT "v=spf1 ip4:192.168.1.80 -all"

# Restart BIND9:
sudo systemctl restart bind9
 
Verify SPF record is live: 
nslookup -type=TXT hasibulwilproject.com 192.168.1.1 
 

Block dangerous attachment types
Configure Postfix to reject emails carrying executable attachment types such as .sh, .exe, .bat, .js, and .ps1 before they reach the mailbox.

Open main.cf on UbuntuServer: 
sudo nano /etc/postfix/main.cf 

Scroll to the very bottom and add this line: 

home_mailbox = Maildir/ 
virtual_alias_maps = hash:/etc/postfix/virtual
 mime_header_checks = regexp:/etc/postfix/mime_checks  —------> ADD HERE 
body_checks = regexp:/etc/postfix/mime_checks 
header_checks = regexp:/etc/postfix/mime_checks 
 

Create the mime_checks file: 
sudo nano /etc/postfix/mime_checks

Add this content to the file: 
/Content-Type:.*name=".*\.(sh|exe|bat|js|ps1)"/i    REJECT Dangerous attachment blocked
/Content-Disposition:.*filename=".*\.(sh|exe|bat|js|ps1)"/i    REJECT Dangerous attachment blocked




 

Restart Postfix: 
sudo systemctl restart postfix
Test it is working:
Try sending the malware attachment again from UbuntuServer:
 

The defense is now working perfectly. ** 550 5.7.1 Dangerous attachment blocked
●	Postfix detected the .sh attachment
●	Rejected the email before it reached the mailbox
●	Connection was closed immediately
Check Thunderbird to confirm the email did NOT arrive in the inbox . No new email arrived.
 



Warning banner on incoming emails:

Open main.cf: 
sudo nano /etc/postfix/main.cf 
header_checks = regexp:/etc/postfix/header_checks 
 
Create the header_checks file: 
/^Subject:/    PREPEND X-Warning: CAUTION - This email may contain phishing content. Do not click suspicious links or open unexpected attachments.
 

Restart Postfix:
sudo systemctl restart postfix 
Send a test email: 
 
swaks --to desktop-user@hasibulwilproject.com \ --from test@hasibulwilproject.com \ --server 192.168.1.80 \ --port 25 \ --subject "Test warning banner" \ --body "Check your email headers" 

Then check Thunderbird More > View Source for the X-Warning line. 

 
The X-Warning banner is working perfectly. 
X-Warning: CAUTION - This email may contain phishing content. Do not click suspicious links or open 

User awareness training
Technical controls alone are not enough. Train users to never click links or open attachments from unexpected emails — even if the sender appears to be internal. Verify suspicious requests by phone before acting.

This sprint demonstrated how spoofed phishing emails with malicious attachments can be delivered through an unprotected Postfix/Dovecot mail server using swaks, exposing critical vulnerabilities in email trust and attachment handling. The attack was documented under MITRE ATT&CK T1566.001.
Four layered defences were implemented and verified: attachment blocking via mime_header_checks, SPF records to prevent sender spoofing, open relay restrictions to limit mail server access, and warning banners to alert recipients of suspicious content. Together these controls significantly reduce the email attack surface and reflect industry-standard SOC defensive practice.
