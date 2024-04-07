# Update DiskStation Certificate

This playbook allows for the DiskStation WebStation SSL certificate to be updated using Ansible Semaphore.
It does not have direct access to the certificate on the NAS, however it uploads the certificate bundle to the `semaphore` users home directory.
The `root` user on the NAS then runs a scheduled task to check if the certificates are present, and moves them if necessary.

## DiskStation Setup

In the **Control Panel** under **Security** > **Certificate** we first need to add our own certificate and set it as system default.  
The initial certificate can be any certificate (self-signed or generated with certbot or similar).

After adding the certificate, open Settings and change `System default` to our newly added certificate.

Once this is done, we need to get the ID of this certificate reference.  
This can be done by SSHing onto the NAS and reading the `/usr/syno/etc/certificate/_archive/INFO` file.
A similar output like the following with show and the ID we want is tied to the service `DSM Desktop Service`.  
Make a note of the ID as we need it for the Scheduled task later.

```json
{
  "YOUR_CERTIFICATE_ID_HERE": {
    // ^ This is the ID we want
    "desc": "",
    "services": [
      {
        "display_name": "DSM Desktop Service",
        "display_name_i18n": "common:web_desktop",
        "isPkg": false,
        "multiple_cert": true,
        "owner": "root",
        "service": "default",
        "subscriber": "system",
        "user_setable": true
      }
    ],
    "user_deletable": true
  }

  // ... REDACTED ...
}
```

### Certificate update task

The following task is added to the Task Scheduler running as the `root` user.  
It is set to run **daily** at a time that makes sense to you.

**NOTE**: Update the `CERT_ID` variable with the ID you got from the previous step.

```bash
#!/bin/bash

CERT_ID=YOUR_CERTIFICATE_ID_HERE

ls /var/services/homes/semaphore/{cert,fullchain,privkey}.pem || exit 1

mv \
    /var/services/homes/semaphore/{cert,fullchain,privkey}.pem \
    "/usr/syno/etc/certificate/_archive/${CERT_ID}"

synosystemctl restart nginx
```

The DiskStation is now ready for Ansible Semaphore to update the certificates.

## Ansible Semaphore Setup

### Key Store

Add a Key Store for decrypting the vault.  
Name: `GitHub Vault` (or whatever makes sense to you)  
Type: `Login with password`  
Login: `<LEAVE BLANK>`  
Password: `<YOUR VAULT PASSWORD>`

### Environment

Add an environment (leave defaults)  
Name: `None`  
Extra variables: `{}`  
Environment variables: `<EMPTY>`

### Inventory

Add an inventory file  
Name: `NAS`
User Credentials: `None`  
Type: `Static`

```ini
[nas]
my-nas.example.com
```

### Repository

Add the GitHub repository  
Name: `GitHub` (or whatever makes sense to you)  
URL: `https://github.com/husjon/ansible-semaphore.git`  
Branch: `main`  
Access Key: `None`

### Task Templates

Name: `Update DiskStation certificate`  
Playbook Filename: `update-diskstation-certificate/playbook.yaml`  
Inventory: `NAS`  
Repository: `GitHub`  
Environment: `None`  
Vault Password: `GitHub Vault`  
Cron: `0 */4 * * *` run the playbook every 4 hours every  
CLI Args:

```json
["-e", "@update-diskstation-certificate/vault.yaml"]
```
