---
title: Automatically provision devices with DPS using X.509 certificates - Azure IoT Edge | Microsoft Docs 
description: Use X.509 certificates to test automatic device provisioning for Azure IoT Edge with Device Provisioning Service
author: kgremban
manager: philmea
ms.author: kgremban
ms.reviewer: mrohera
ms.date: 10/04/2019
ms.topic: conceptual
ms.service: iot-edge
services: iot-edge
---

# Create and provision an IoT Edge device using X.509 certificates (Preview)

|Important |
|--------- |
|The information in this preview article is applicable to installations of IoTEdge [1.0.9-rc2](https://github.com/Azure/azure-iotedge/releases/tag/1.0.9-rc2) or later. |

Azure IoT Edge devices can be auto-provisioned using the [Device Provisioning Service](https://docs.microsoft.com/azure/iot-dps/index.yml) just like devices that aren't edge-enabled. If you're unfamiliar with the process of auto-provisioning, review the [auto-provisioning concepts](https://docs.microsoft.com/azure/iot-dps/concepts-auto-provisioning) before continuing.

This article shows you how to create a Device Provisioning Service individual enrollment using X.509 certificates on an IoT Edge device with the following steps:

* Create an instance of IoT Hub Device Provisioning Service (DPS).
* Generate certificates and keys.
* Create an individual enrollment for the device.
* Install the IoT Edge runtime and connect to the IoT Hub.

Using X.509 certificates as an attestation mechanism is an excellent way to scale production and simplify device provisioning. X.509 certificates are typically arranged in a certificate chain of trust in which each certificate in the chain is signed by the private key of the next higher certificate, and so on, terminating in a self-signed root certificate or a trusted root certificate. This arrangement establishes a delegated chain of trust from the root certificate generated by a trusted root certificate authority (CA) down through each intermediate CA to the end-entity "leaf" certificate installed on a device.

## Prerequisites

* An active IoT Hub
* A physical or virtual device
* The latest version of [Git](https://git-scm.com/download/) installed.

## Set up the IoT Hub Device Provisioning Service

Create a new instance of the IoT Hub Device Provisioning Service in Azure, and link it to your IoT hub. You can follow the instructions in [Set up the IoT Hub DPS](https://docs.microsoft.com/azure/iot-dps/quick-setup-auto-provision).

After you have the Device Provisioning Service running, copy the value of **ID Scope** from the overview page. You use this value when you configure the IoT Edge runtime.

## Generate certificates with Linux

Use the steps in this section to generate test certificates on Linux. You can use a Linux machine to generate the certificates, and then use them when creating a deployment for any IoT Edge device running on any supported operating system.

The certificates generated in this section are intended for testing purposes only and should not be used in a production environment.

### Prepare creation scripts

The Azure IoT Edge git repository contains scripts that you can use to generate test certificates. In this section, you clone the IoT Edge repo and execute the scripts.

1. Clone the git repo that contains scripts to generate non-production certificates. These scripts help you create the necessary certificates to use X.509 attestation when creating a DPS enrollment.

   ```bash
   git clone https://github.com/Azure/iotedge.git
   ```

1. Navigate to the directory in which you want to work. We'll refer to this directory throughout the article as *\<WRKDIR>*. All certificate and key files will be created in this directory.
  
1. Copy the config and script files from the cloned IoT Edge repo into your working directory.

   ```bash
   cp <path>/iotedge/tools/CACertificates/*.cnf .
   cp <path>/iotedge/tools/CACertificates/certGen.sh .
   ```

### Create certificates

In this section, you create the certificate and key files that you'll use later on when creating your DPS enrollment and installing the IoT Edge runtime.

1. Create the root CA certificate and one intermediate certificate. These certificates are placed in *\<WRKDIR>*.

   If you've already created root and intermediate certificates in this working directory, don't run this script again. Rerunning this script will overwrite the existing certificates. Instead, continue with the next step.

   ```bash
   ./certGen.sh create_root_and_intermediate
   ```

   The script creates several certificates and keys. If you're creating a group enrollment, make note of one, which we'll refer to when adding your enrollment:

   `<WRKDIR>/certs/azure-iot-test-only.root.ca.cert.pem`

1. Create the IoT Edge device identity certificate and private key with the following command. Replace `<name>` with your preferred device ID.

   ```bash
   ./certGen.sh create_edge_device_identity_certificate "<name>"
   ```

   The script creates several certificates and keys, including two that you'll use when creating an individual enrollment and installing the IoT Edge runtime:
   * `<WRKDIR>/certs/iot-edge-device-identity-<name>.cert.pem`
   * `<WRKDIR>/private/iot-edge-device-identity-<name>.key.pem`

   > [!NOTE]
   > These certificates and keys are used only for DPS enrollments, and shouldn't be confused with the CA certificates that the IoT Edge device presents to modules or leaf devices for verification. For more information, see [Azure IoT Edge certificate usage detail](https://docs.microsoft.com/azure/iot-edge/iot-edge-certs).

## Generate certificates with Windows

Use the steps in this section to generate test certificates on Windows. You can use a Windows machine to generate the certificates, and then use them when creating a deployment for any IoT Edge device running on any supported operating system.

The certificates generated in this section are intended for testing purposes only and should not be used in a production environment.

### Install OpenSSL

Install OpenSSL for Windows on the machine that you're using to generate the certificates. If you already have OpenSSL installed on your Windows device, you may skip this step, but ensure that openssl.exe is available in your PATH environment variable.

There are several ways you can install OpenSSL:

* **Easier:** Download and install any [third-party OpenSSL binaries](https://wiki.openssl.org/index.php/Binaries), for example, from [OpenSSL on SourceForge](https://sourceforge.net/projects/openssl/). Add the full path to openssl.exe to your PATH environment variable.

* **Recommended:** Download the OpenSSL source code and build the binaries on your machine by yourself or via [vcpkg](https://github.com/Microsoft/vcpkg). The instructions listed below use vcpkg to download source code, compile, and install OpenSSL on your Windows machine with easy steps.

   1. Navigate to a directory where you want to install vcpkg. We'll refer to this directory as *\<VCPKGDIR>*. Follow the instructions to download and install [vcpkg](https://github.com/Microsoft/vcpkg).

   1. Once vcpkg is installed, run the following command from a PowerShell prompt to install the OpenSSL package for Windows x64. The installation typically takes about 5 mins to complete.

      ```powershell
      .\vcpkg install openssl:x64-windows
      ```

   1. Add `<VCPKGDIR>\installed\x64-windows\tools\openssl` to your PATH environment variable so that the openssl.exe file is available for invocation.

### Prepare creation scripts

The Azure IoT Edge git repository contains scripts that you can use to generate test certificates. In this section, you clone the IoT Edge repo and execute the scripts.

1. Open a PowerShell window in administrator mode.

1. Clone the git repo that contains scripts to generate non-production certificates. These scripts help you create the necessary certificates to use X.509 attestation when creating a DPS enrollment. Use the `git clone` command or [download the ZIP](https://github.com/Azure/iotedge/archive/master.zip).

   ```powershell
   git clone https://github.com/Azure/iotedge.git
   ```

1. Navigate to the directory in which you want to work. Throughout this article, we'll call this directory *\<WRKDIR>*. All certificates and keys will be created in this working directory.

1. Copy the configuration and script files from the cloned repo into your working directory.

   ```powershell
   copy <path>\iotedge\tools\CACertificates\*.cnf .
   copy <path>\iotedge\tools\CACertificates\ca-certs.ps1 .
   ```

   If you downloaded the repo as a ZIP, then the folder name is `iotedge-master` and the rest of the path is the same.

1. Enable PowerShell to run the scripts.

   ```powershell
   Set-ExecutionPolicy -ExecutionPolicy Unrestricted -Scope CurrentUser
   ```

1. Bring the functions used by the scripts into PowerShell's global namespace.

   ```powershell
   . .\ca-certs.ps1
   ```

   The PowerShell window will display a warning that the certificates generated by this script are only for testing purposes, and should not be used in production scenarios.

1. Verify that OpenSSL has been installed correctly and make sure that there won't be name collisions with existing certificates. If there are problems, the script should describe how to fix them on your system.

   ```powershell
   Test-CACertsPrerequisites
   ```

### Create certificates

In this section, you create the certificate and key files that you'll use later on when creating your DPS enrollment and installing the IoT Edge runtime.

1. Create the root CA certificate and have it sign one intermediate certificate. The certificates are all placed in your working directory.

   If you've already created root and intermediate certificates in this working directory, don't run this script again. Rerunning this script will overwrite the existing certificates. Instead, continue with the next step.

   ```powershell
   New-CACertsCertChain rsa
   ```

   This script command creates several certificates and keys. If you're creating a group enrollment, make note of one, which we'll refer to when adding your enrollment:

   `<WRKDIR>\certs\azure-iot-test-only.root.ca.cert.pem`

1. Create the IoT Edge device identity certificate and private key with the following command. Replace `<name>` with your preferred device ID.

   ```powershell
   New-CACertsEdgeDeviceIdentity  "<name>"
   ```

   This command creates several certificate and key files, including two that you'll use when creating an individual enrollment and installing the IoT Edge runtime:
   * `<WRKDIR>\certs\iot-edge-device-identity-<name>.cert.pem`
   * `<WRKDIR>\private\iot-edge-device-identity-<name>.key.pem`

   > [!NOTE]
   > These certificates and keys are used only for DPS enrollments, and shouldn't be confused with the CA certificates that the IoT Edge device presents to modules or leaf devices for verification. For more information, see [Azure IoT Edge certificate usage detail](https://docs.microsoft.com/azure/iot-edge/iot-edge-certs).

## Create a DPS enrollment

Use your generated certificates and keys to create an individual enrollment in DPS.

When you create an enrollment in DPS, you have the opportunity to declare an **Initial Device Twin State**. In the device twin, you can set tags to group devices by any metric you need in your solution, like region, environment, location, or device type. These tags are used to create [automatic deployments](https://docs.microsoft.com/azure/iot-edge/how-to-deploy-monitor).

> [!TIP]
> Group enrollments are also possible when using X.509 certificates and involve many of the same decisions as individual enrollments. When creating a group enrollment, you also have the option of using either root CA certificates (such as the `<WRKDIR>\certs\azure-iot-test-only.root.ca.cert.pem` certificate) or intermediate certificates.

1. In the [Azure portal](https://portal.azure.com), navigate to your instance of IoT Hub Device Provisioning Service.

1. Under **Settings**, select **Manage enrollments**.

1. Select **Add individual enrollment** then complete the following steps to configure the enrollment:  

   1. For **Mechanism**, select **X.509**.

   1. Upload a **Primary Certificate .pem or .cer file** by selecting the device identity certificate you created earlier:

      `<WRKDIR>/certs/iot-edge-device-identity-<name>.cert.pem`

   1. Provide an **IoT Hub Device ID** for your device if you'd like. You can use device IDs to target an individual device for module deployment. If you don't provide a device ID, the common name (CN) in the X.509 certificate is used.

   1. Select **True** to declare that the enrollment is for an IoT Edge device. For a group enrollment, all devices must be IoT Edge devices or none of them can be.

   1. Accept the default value from the Device Provisioning Service's allocation policy for **how you want to assign devices to hubs** or choose a different value that is specific to this enrollment.

   1. Choose the linked **IoT Hub** that you want to connect your device to. You can choose multiple hubs, and the device will be assigned to one of them according to the selected allocation policy.

   1. Choose **how you want device data to be handled on re-provisioning** when devices request provisioning after the first time.

   1. Add a tag value to the **Initial Device Twin State** if you'd like. You can use tags to target groups of devices for module deployment. For example:

      ```json
      {
         "tags": {
            "environment": "test"
         },
         "properties": {
            "desired": {}
         }
      }
      ```

   1. Ensure **Enable entry** is set to **Enable**.

   1. Select **Save**.

Now that an enrollment exists for this device, the IoT Edge runtime can automatically provision the device during installation.

## Install the IoT Edge runtime

The IoT Edge runtime is deployed on all IoT Edge devices. Its components run in containers, and allow you to deploy additional containers to the device so that you can run code at the edge.

You'll need the following information when provisioning your device:

* The DPS **ID Scope** value
* The URI to the device identity certificate in the format `file:///path/identity_certificate.pem`
* The URI to the device identity key in the format `file:///path/identity_key.pem`
* An optional registration ID (pulled from the common name in the device identity certificate if not supplied)

### Linux device

Follow the instructions for your device's architecture. Make sure to configure the IoT Edge runtime for automatic, not manual, provisioning.

[Install the Azure IoT Edge runtime on Linux](https://docs.microsoft.com/azure/iot-edge/how-to-install-iot-edge-linux)

The section in the configuration file for X.509 provisioning looks like this:

```yaml
# DPS X.509 provisioning configuration
 provisioning:
   source: "dps"
   global_endpoint: "https://global.azure-devices-provisioning.net"
   scope_id: "{scope_id}"
   attestation:
     method: "x509"
     registration_id: "<OPTIONAL REGISTRATION ID. IF UNSPECIFIED CAN BE OBTAINED FROM CN OF identity_cert"
     identity_cert: "<REQUIRED URI TO DEVICE IDENTITY CERTIFICATE>"
     identity_pk: "<REQUIRED URI TO DEVICE IDENTITY PRIVATE KEY>"
```

Replace the placeholder values for `scope_id`, `identity_cert`, `identity_pk`, and optionally `registration_id` with the data you collected earlier.

### Windows device

Install the IoT Edge runtime on the device for which you generated the identity certificate and identity key. You'll configure the IoT Edge runtime for automatic, not manual, provisioning.

For more detailed information about installing IoT Edge on Windows, including prerequisites and instructions for tasks like managing containers and updating IoT Edge, see [Install the Azure IoT Edge runtime on Windows](https://docs.microsoft.com/azure/iot-edge/how-to-install-iot-edge-windows).

1. Open a PowerShell window in administrator mode. Be sure to use an AMD64 session of PowerShell when installing IoT Edge, not PowerShell (x86).

1. The **Deploy-IoTEdge** command checks that your Windows machine is on a supported version, turns on the containers feature, and then downloads the moby runtime and the IoT Edge runtime. The command defaults to using Windows containers.

   ```powershell
   . {Invoke-WebRequest -useb https://aka.ms/iotedge-win} | Invoke-Expression; `
   Deploy-IoTEdge
   ```

1. At this point, IoT Core devices may restart automatically. Other Windows 10 or Windows Server devices may prompt you to restart. If so, restart your device now. Once your device is ready, run PowerShell as an administrator again.

1. The **Initialize-IoTEdge** command configures the IoT Edge runtime on your machine. The command defaults to manual provisioning with Windows containers unless you use the `-Dps` flag to use automatic provisioning.

   Replace the placeholder values for `{scope_id}`, `{identity cert URI}`, and `{identity key URI}` with the data you collected earlier. If you want to specify the registration ID, include `-RegistrationId {registration_id}` as well, replacing the placeholder as appropriate.

   ```powershell
   . {Invoke-WebRequest -useb https://aka.ms/iotedge-win} | Invoke-Expression; `
   Initialize-IoTEdge -Dps -ScopeId {scope ID} -X509IdentityCertificate {identity cert URI} -X509IdentityPrivateKey {identity key URI}
   ```

## Verify successful installation

If the runtime started successfully, you can go into your IoT Hub and start deploying IoT Edge modules to your device. Use the following commands on your device to verify that the runtime installed and started successfully.

### Linux device

Check the status of the IoT Edge service.

```cmd/sh
systemctl status iotedge
```

Examine service logs.

```cmd/sh
journalctl -u iotedge --no-pager --no-full
```

List running modules.

```cmd/sh
iotedge list
```

### Windows device

Check the status of the IoT Edge service.

```powershell
Get-Service iotedge
```

Examine service logs.

```powershell
. {Invoke-WebRequest -useb aka.ms/iotedge-win} | Invoke-Expression; Get-IoTEdgeLog
```

List running modules.

```powershell
iotedge list
```

You can verify that the individual enrollment that you created in Device Provisioning Service was used. Navigate to your Device Provisioning Service instance in the Azure portal. Open the enrollment details for the individual enrollment that you created. Notice that the status of the enrollment is **assigned** and the device ID is listed.

## Next steps

The Device Provisioning Service enrollment process lets you set the device ID and device twin tags at the same time as you provision the new device. You can use those values to target individual devices or groups of devices using automatic device management. Learn how to [Deploy and monitor IoT Edge modules at scale using the Azure portal](https://docs.microsoft.com/azure/iot-edge/how-to-deploy-monitor) or [using Azure CLI](https://docs.microsoft.com/azure/iot-edge/how-to-deploy-monitor-cli).