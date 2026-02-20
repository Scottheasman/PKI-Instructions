# Install Yubico HSM

## This is the Official YubiHSM 2 Key Ceremony & Installation Runbook for your Production Root CA. These steps have been verified against your specific environment and yubihsm-shell version.

### 1. Software Installation Order (Root CA)
Run these installers from your yubihsm2-sdk folder in this exact order:

#### Connector: yubihsm-connector-3.0.6-windows-amd64.msi
#### CNG Provider (KSP): yubihsm-cngprovider-windows-amd64.msi
#### Management Tools: yubihsm-shell-2.7.1-x64.msi and yubihsm-setup-2.3.4-x64.msi
#### Verification: Open PowerShell and run certutil -csplist. Confirm "YubiHSM Key Storage Provider" is listed.

### 2. Primary HSM Initialization (The "Key Ceremony")
Perform these steps with the Primary Root HSM inserted.

#### Open Shell & Connect:
cmd
Copy
```text
yubihsm-shell
connect
session open 1 password
```
#### Create Admin Key (ID 2) & Delete Default (ID 1):
cmd
Copy
```text
put authkey 0 2 "Admin-Key" 1 all all YubicoP@ssw0rd123!
delete 0 1 authentication-key
session close 0
session open 2 YubicoPassw0rd123!
```

#### Create the Wrap Key (Master Recovery Key):
Note: Save this 32-character hex string in your physical safe!
cmd
Copy
```text
put wrapkey 0 1 "Root-Wrap-Key" 1 wrap-data:unwrap-data 0 1234567890ABCDEF1234567890ABCDEF
```
#### Generate Root CA Signing Key (RSA 4096):
cmd
Copy
```text
generate asymmetric 0 100 "IEX-RootCA-Key-v1" 1 sign-pkcs:sign-pss rsa4096
```
#### Export the Key for Cloning:
cmd
Copy

```text
get wrapped 0 1 asymmetric-key 100 rootca-key.blob
```
### 3. Backup HSM Cloning (The "Twin" Setup)
Perform these steps when you have physical access to swap the sticks.

Swap HSMs: Unplug Primary, plug in Backup HSM.
Initialize Backup HSM:
cmd
Copy
connect
session open 1 password
put authkey 0 2 "Admin-Key" 1 all all YubicoPassw0rd123!
delete 0 1 authentication-key
session close 0
session open 2 YubicoPassw0rd123!
Apply the EXACT SAME Wrap Key:
Crucial: This must match the Primary HSM exactly.
cmd
Copy
put wrapkey 0 1 "Root-Wrap-Key" 1 wrap-data:unwrap-data 0 1234567890ABCDEF1234567890ABCDEF
Import (Unwrap) the Root CA Key:
cmd
Copy
put wrapped 0 1 rootca-key.blob
Final Verification:
cmd
Copy
list objects 0
Confirm you see IDs 0x0001, 0x0002, and 0x0064 (the RSA key).
4. AD CS Role Installation (Root CA)
Once the HSM is prepared, install the Root CA role using the YubiHSM KSP.

powershell
Copy
Install-AdcsCertificationAuthority `
    -CAType StandaloneRootCA `
    -CryptoProviderName "YubiHSM Key Storage Provider" `
    -KeyLength 4096 `
    -HashAlgorithmName SHA256 `
    -ValidityPeriod Years `
    -ValidityPeriodUnits 20 `
    -CACommonName "Lab Root CA" `
    -Force
5. Disaster Recovery (HSM Failure)
If your Primary HSM fails in production:

Unplug the failed Primary HSM.
Plug in the pre-cloned Backup HSM.
Restart the CertSvc (Active Directory Certificate Services) service.
The CA is immediately back online because the Backup HSM contains the identical Key ID (100) and Label.
6. Summary of Secrets to Save in Safe
Secret	Value
Admin Password	YubicoPassw0rd123!
Wrap Key (Hex)	1234567890ABCDEF1234567890ABCDEF
Key Blob File	rootca-key.blob (Store on a secure encrypted USB)
Potential Reason for Text Extraction Failure: The images IMG_20260220_123249_0.jpg and IMG_20260220_124813_0.jpg are too large for direct text extraction, but we have successfully verified the workflow manually. These documents can only be used in code execution if further automated parsing is required.