# Using IBM-managed OS licenses for migrated RHEL and Windows VSIs on IBM Cloud VPC

When you bring an operating system to IBM Cloud, whether you import your own image or use live migration, you usually bring your own license too.

This guide covers the other option: licensing that operating system through an **IBM-managed subscription**, so IBM Cloud provides the license instead. It covers **RHEL** and **Windows Server** on IBM Cloud VPC Virtual Server Instances (VSIs), and works with any migration method, including **Bring Your Own Image (BYOI)** and **live migration** with tools such as RackWare.

> [!NOTE]
> This is a community guide, not official IBM documentation. Last tested: November 2026. 

## Overview

RHEL and Windows are licensed differently:

| OS      | Activation method                                    |
|---------|------------------------------------------------------|
| RHEL    | Automatic registration through cloud-init            |
| Windows | Manual activation (`slmgr`) with a GVLK against IBM's KMS server |

### Tested environments

**RHEL** (VSI profile: `cx3d-2x5`)

| RHEL version | cloud-init version |
|--------------|--------------------|
| 8.10         | 23.4-7.el8_10.11   |
| 9.6          | 24.4-4.el9_6.5     |
| 9.8          | 24.4-8.el9_8.1     |
| 10.0         | 24.4-3.el10_0.4    |
| 10.2         | 24.4-7.el10_2.1    |

**Windows Server** (VSI profile: `cx3d-4x10`)

| Windows Server version | Cloudbase-Init version |
|------------------------|------------------------|
| 2016 Standard          | 1.1.2                  |
| 2019 Standard          | 1.1.2                  |
| 2022 Standard          | 1.1.2                  |
| 2025 Standard          | 1.1.6                  |

> [!WARNING]
> IBM may change the subscription process for RHEL and Windows in the future, including the cloud-init and KMS configuration. If this procedure stops working, contact IBM Cloud Support.

# Before you start

Make sure that:

* Your target operating system is supported on IBM Cloud VPC. See [About images](https://cloud.ibm.com/docs/vpc?topic=vpc-about-images).
* cloud-init (RHEL) or Cloudbase-Init (Windows) is installed.
* The required virtio drivers are installed.
* The VSI can reach the IBM licensing endpoints (see [Troubleshooting](#troubleshooting)).

For more information, see [Creating a Linux custom image](https://cloud.ibm.com/docs/vpc?topic=vpc-create-linux-custom-image) and [Creating a Windows custom image](https://cloud.ibm.com/docs/vpc?topic=vpc-create-windows-custom-image).

### If you import a custom image

When you import the image, choose the operating system **without** the `-byol` suffix (for example, `windows-2022-amd64` instead of `windows-2022-amd64-byol`). This tells IBM Cloud to provide the license.

### If you use live migration (for example, RackWare)

Provision the target VSI from a stock image, or from a custom image without `-byol`, so that IBM Cloud provides the license.

# Licensing RHEL

On RHEL, cloud-init handles the whole registration, as described in [Troubleshooting RHEL subscription issues](https://developer.ibm.com/tutorials/awb-troubleshoot-rhel-subscription-issues/).

### 1. Rerun cloud-init

After the migration, clear the cloud-init state and reboot so the registration runs again:

```bash
sudo cloud-init clean --logs --reboot
```

> [!CAUTION]
> `cloud-init clean` makes cloud-init treat the next boot as a first boot. It runs all per-instance modules again, which can regenerate the SSH host keys and reapply hostname, user, and SSH key settings from the instance metadata. On a migrated server, check that this won't overwrite configuration you need to keep, and expect SSH clients to warn about changed host keys.

### 2. Check the registration status

```bash
sudo subscription-manager status
```

Expected output:

```text
+-------------------------------------------+
   System Status Details
+-------------------------------------------+
Overall Status: Registered
```

### 3. Check which server the system is registered against

```bash
sudo subscription-manager config --list | grep -E "hostname|baseurl|prefix|port"
```

Expected output (example for `eu-es`):

```text
   hostname = rhha02.updates.eu-es.iaas.service.networklayer.com
   port = 8443
   prefix = /rhsm
   proxy_hostname = []
   proxy_port = []
   baseurl = https://rhha02.updates.eu-es.iaas.service.networklayer.com/pulp/repos
```

### How it works

* IBM Cloud attaches a cloud-init data source to the VSI with the subscription details: activation key, organization, and a few related settings.
* During provisioning, cloud-init reads those details and registers the system with `subscription-manager`.

![Diagram of the cloud-init registration flow: IBM Cloud provides the subscription details through the cloud-init data source, and cloud-init registers the VSI against IBM's Red Hat Satellite servers](cloudinit.avif)

# Licensing Windows

On Windows, you activate the license against IBM's KMS server (`kms.adn.networklayer.com:1688`) using Microsoft's public KMS client keys (GVLKs).

After the migration, run these commands from an **elevated** PowerShell or Command Prompt.

> [!TIP]
> `slmgr` shows its output in pop-up windows. To get the output in the console, run it through `cscript`, for example `cscript //nologo C:\Windows\System32\slmgr.vbs /dli`.

### 1. Set the KMS server

```powershell
slmgr /skms kms.adn.networklayer.com:1688
```

### 2. Install the GVLK for your edition

Find the key for your edition in Microsoft's [KMS client activation keys](https://learn.microsoft.com/en-us/windows-server/get-started/kms-client-activation-keys). For example, for Windows Server 2025 Standard:

```powershell
slmgr /ipk TVRH6-WHNXV-R9WG3-9XRFY-MY832
```

### 3. Activate

```powershell
slmgr /ato
```

### 4. Check the license status

```powershell
slmgr /dli
```

The output should include `License Status: Licensed`.

### How it works

A GVLK (Generic Volume License Key) is a public product key published by Microsoft. It tells Windows to activate against a KMS server instead of Microsoft's online activation service. Here, that KMS server is IBM's, so the license is provided and billed through IBM Cloud.

## Summary

Whether you import your own image or use live migration, you can license RHEL and Windows on IBM Cloud VPC through an IBM-managed subscription instead of bringing your own license. The key is to avoid the `-byol` operating system as the base image, so that IBM Cloud knows it must provide the license.

* **RHEL**: cloud-init registers the system automatically against IBM's Red Hat Satellite servers. After a migration, run `sudo cloud-init clean --logs --reboot` to trigger the registration again, and check the result with `subscription-manager status`.
* **Windows**: point Windows to IBM's KMS server (`kms.adn.networklayer.com:1688`), install the GVLK for your edition, and activate with `slmgr /ato`. Don't rely on Cloudbase-Init for this: it only knows the keys up to Windows Server 2016.

## Troubleshooting

### Check connectivity to the licensing endpoints

RHEL (replace `<region>`):

```bash
curl -v https://rhha01.updates.<region>.iaas.service.networklayer.com:8443/rhsm
```

Windows:

```powershell
Test-NetConnection kms.adn.networklayer.com -Port 1688
```

`TcpTestSucceeded` must be `True`.

### Windows activation fails with error 0xC004F074

This error means Windows couldn't reach the KMS server. Check the connectivity test above, your security groups and network ACLs, and that the KMS server is set correctly (`slmgr /dlv` shows the configured KMS name).

## Appendix

### 1. Why you can't rely on Cloudbase-Init to activate Windows

Cloudbase-Init can set the KMS client key for you, but it takes the key from the list in [productkeys.py](https://github.com/cloudbase/cloudbase-init/blob/master/cloudbaseinit/utils/windows/productkeys.py). That list only goes up to Windows Server 2016, so activation fails for any newer version. In our tests, Windows Server 2016 Standard activated correctly and newer versions did not.

> [!NOTE]
> This method works only for Windows Server 2016.

To use it on Windows Server 2016:

1. Add `set_kms_product_key=true` to both `cloudbase-init.conf` and `cloudbase-init-unattend.conf`.
2. Rerun Cloudbase-Init (see [Force Cloudbase-Init to run again](#force-cloudbase-init-to-run-again)).
3. Wait about a minute, then check the license status:

   ```powershell
   slmgr /dli
   ```

### 2. Force Cloudbase-Init to run again
```powershell
Remove-Item -Path "HKLM:\SOFTWARE\Cloudbase Solutions\Cloudbase-Init\$id" -Recurse
Restart-Service cloudbase-init
```

### 3. Cloudbase-Init configuration path

```text
C:\Program Files\Cloudbase Solutions\Cloudbase-Init\conf\cloudbase-init.conf
```

### 4. Remove the Windows license

Uninstall the current product key, which leaves Windows unlicensed:

```powershell
slmgr /upk
```

Clear the product key from the registry so it can't be read or reused (for example, by malicious software):

```powershell
slmgr /cpky
```

Clear the KMS server name and port from the registry, so Windows goes back to finding a KMS host through DNS:

```powershell
slmgr /ckms
```

### 5. Show the Windows license status

Basic license information:
```bash
slmgr /dli
```
Detailed license information:
```bash
slmgr /dlv
```  
Shows whether Windows is permanently activated or when the activation expires:
```bash
slmgr /xpr
```

# References

* [IBM Cloud VPC: About images](https://cloud.ibm.com/docs/vpc?topic=vpc-about-images)
* [IBM Cloud VPC: Creating a Windows custom image](https://cloud.ibm.com/docs/vpc?topic=vpc-create-windows-custom-image)
* [IBM Cloud VPC: Creating a Linux custom image](https://cloud.ibm.com/docs/vpc?topic=vpc-create-linux-custom-image)
* [IBM Developer: Troubleshooting RHEL subscription issues](https://developer.ibm.com/tutorials/awb-troubleshoot-rhel-subscription-issues/)
* [cloud-init source code](https://github.com/canonical/cloud-init/tree/main/cloudinit)
* [Cloudbase-Init source code](https://github.com/cloudbase/cloudbase-init)
* [Microsoft: KMS client activation keys](https://learn.microsoft.com/en-us/windows-server/get-started/kms-client-activation-keys)
