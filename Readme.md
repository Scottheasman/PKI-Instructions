## Templates

### IEX-CESCEP-Computer

This is a robust "modern computer" template designed for autoenrollment of domain-joined Windows machines via CES/CEP.

#### General Tab
Validity: 1 Year / 6 Weeks.
Publish certificate in AD: Unchecked.
Accuracy: Correct. Computer certificates are typically not published to AD to prevent database bloat; they are managed locally by the machine.

#### Compatibility Tab
Settings: Windows Server 2012 R2 / Windows 8.1.
Accuracy: Perfect. This enables modern KSP and SHA-256 while maintaining compatibility with all modern Windows versions.

#### Cryptography Tab
Provider Category: Key Storage Provider (KSP).
Key Size: 4096.
Providers: Microsoft Software Key Storage Provider.
Accuracy: Excellent. High security for machine identity using the modern software KSP.

#### Subject Name Tab
Subject Name Format: Common Name.
Include in SAN: DNS Name, Service Principal Name (SPN).
Accuracy: Perfect. Adding the SPN ensures compatibility with modern services that verify machine identity via the SAN field.
#### Extensions Tab
Application Policies: Client Authentication.
Accuracy: Correct. This allows the computer to identify itself to network services and the CES/CEP servers.

#### Security Tab
Permissions: Domain Computers -> Read, Enroll, Autoenroll.
Accuracy: Perfect. This ensures every domain-joined machine automatically receives its identity certificate without manual intervention.

### IEX-CESCEP-User
This is a robust "modern user" template designed for autoenrollment of domain users via CES/CEP, supporting S/MIME, EFS, and Client Authentication.

#### General Tab
Validity: 1 Year / 6 Weeks.
Publish certificate in Active Directory: Checked.
Accuracy: Correct. Unlike computer certificates, it is standard practice to publish user certificates to AD. This allows other users to find your certificate (e.g., for sending encrypted emails via S/MIME).

#### Compatibility Tab
Settings: Windows Server 2012 R2 / Windows 8.1.
Accuracy: Perfect. This ensures the user gets a modern KSP-based certificate with SHA-256 support.

#### Cryptography Tab
Provider Category: Key Storage Provider (KSP).
Key Size: 4096.
Providers: Microsoft Software Key Storage Provider.
Accuracy: Excellent. This provides very high security for a user identity certificate.

#### Subject Name Tab
Subject Name Format: Fully distinguished name.
Include in Subject/SAN: E-mail name, User Principal Name (UPN).
Accuracy: Perfect. Including the email in both the Subject and the SAN ensures compatibility with both old and new email clients (Outlook, mobile devices, etc.). The UPN is essential for logging into VPNs or Wi-Fi using the certificate.

#### Extensions Tab
Application Policies: Client Authentication, Secure Email, Encrypting File System (EFS).
Accuracy: Correct. This is a "Swiss Army Knife" user certificate. It allows the user to log in to the network, sign/encrypt emails, and use Windows EFS to encrypt files on their hard drive.

#### Security Tab
Permissions: Domain Users -> Read, Enroll, Autoenroll.
Accuracy: Perfect. This ensures that every user who logs into a domain-joined machine will automatically receive their personal certificate via the CES/CEP services without any manual steps.

### IEX-INFRA-OCSP
This specialized template is for the Online Responder (OCSP) service to sign revocation status responses.

#### General Tab
Validity: 2 Weeks / 2 Days.
Accuracy: Perfect. OCSP signing certs must be short-lived because they contain the "No Revocation Checking" extension and cannot be revoked if compromised.

#### Compatibility Tab
Settings: Windows Server 2016 / Windows 10.
Accuracy: Perfect. Allows for the most modern cryptographic standards.

#### Cryptography Tab
Provider Category: Key Storage Provider (KSP).
Key Size: 4096.
Accuracy: Excellent. High-strength signing for the responder service.

#### Subject Name Tab
Setting: Build from AD / DNS Name.
Accuracy: Correct. The responder uses its own FQDN as the identity for the signing certificate.

#### Extensions Tab
Application Policies: OCSP Signing.
OCSP No Revocation Checking: Present.
Accuracy: Perfect. This extension is mandatory to prevent circular revocation loops.

#### Security Tab
Permissions:
OCSP1,OCSP2 -> Read, Enroll, Autoenroll.
Accuracy: Perfect. Targeted specifically to the computer accounts running the OCSP role.

### IEX-MANUALCSR-EnrollWeb-SSL
This template is designed for manual requests (CSRs) for web servers and appliances that require private key portability.

#### General Tab
Validity: 1 Year / 6 Weeks.
Accuracy: Correct. Standard lifecycle for SSL/TLS certificates.

#### Compatibility Tab
Settings: Windows Server 2016 / Windows 10.
Accuracy: Perfect. Supports modern TLS requirements.

#### Request Handling Tab
Allow private key to be exported: Checked.
Accuracy: Correct. Essential for manual templates so the cert/key can be moved to the target web server or load balancer.

#### Subject Name Tab
Setting: Supply in the request.
Accuracy: Mandatory. SSL certs must match the website URL provided in the CSR, not the server's AD name.

#### Issuance Requirements Tab
CA certificate manager approval: Checked.
Accuracy: Excellent. High-security "human-in-the-loop" workflow for manual requests.

#### Security Tab
Permissions: PKI_WebCert_Requesters -> Read, Enroll.
Accuracy: Perfect. Delegated enrollment rights to a specific security group.

### IEX-NDES-Device
The "workhorse" template for network devices (routers/switches) enrolling via the SCEP protocol.

#### General Tab
Validity: 1 Year / 6 Weeks.
Accuracy: Correct. Standard for network device identity.
#### Cryptography Tab
Provider Category: Key Storage Provider (KSP).
Key Size: 4096.
Accuracy: Excellent. Modern security for network infrastructure.

#### Subject Name Tab
Setting: Supply in the request.
Accuracy: Mandatory. SCEP requires the device to provide its own identity in the request.
#### Extensions Tab
Application Policies: Client Authentication, IP Security IKE Intermediate.
Accuracy: Perfect. Adding IKE Intermediate ensures the cert is valid for VPN/IPsec tunnels on network hardware.

Key Usage Extension
Settings: Digital Signature, Key Encipherment.
Criticality: Marked as Critical.
Accuracy: Mandatory. SCEP is strict; the Key Usage must be marked critical or many routers will reject the cert.

#### Security Tab
Permissions: PKINDESSvc -> Read, Enroll.
Accuracy: Perfect. The NDES service account must have rights to proxy these requests to the CA.

### IEX-NDES-RA
The Registration Authority template used internally by the NDES service. This is the most sensitive configuration in the lab.

#### General Tab
Validity: 1 Year / 6 Weeks.
Accuracy: Correct. Matches the NDES service lifecycle.

#### Compatibility Tab
Settings: Windows Server 2003 / Windows XP.
Accuracy: Perfect. This is the "Magic Fix" that enables the Legacy CSP required by the NDES role.

#### Cryptography Tab
Provider Category: Legacy Cryptographic Service Provider.
Provider: Microsoft Strong Cryptographic Provider.
Accuracy: Mandatory. NDES expects this specific legacy provider during its internal enrollment process.

#### Subject Name Tab
Setting: Supply in the request.
Accuracy: Mandatory. NDES specifies the MSCEP-RA subject name internally.

#### Request Handling Tab
Purpose: Signature and encryption.
Accuracy: Correct. RA certs must be able to both sign SCEP responses and encrypt challenge passwords.

#### Security Tab
Permissions: PKINDESSvc, Domain Computers -> Read, Enroll.
Accuracy: Perfect. Ensures the service account and the NDES server itself can manage these critical certs.

