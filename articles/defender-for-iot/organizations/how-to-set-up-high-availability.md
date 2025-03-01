---
title: Set up high availability
description: Increase the resiliency of your Defender for IoT deployment by installing an on-premises management console high availability appliance. High availability deployments ensure your managed sensors continuously report to an active on-premises management console.
ms.date: 01/24/2023 
ms.topic: how-to
---
# About high availability

Increase the resiliency of your Defender for IoT deployment by configuring high availability on your on-premises management console. High availability deployments ensure your managed sensors continuously report to an active on-premises management console.

This deployment is implemented with an on-premises management console pair that includes a primary and secondary appliance.

> [!NOTE]
> In this document, the principal on-premises management console is referred to as the primary, and the agent is referred to as the secondary.

## About primary and secondary communication

When a primary and secondary on-premises management console is paired:

- An on-premises management console SSL certificate is applied to create a secure connection between the primary and secondary appliances. The SSL may be the self-signed certificate installed by default or a certificate installed by the customer.

    When validation is `ON`, the appliance should be able to establish connection to the CRL server defined by the certificate.

- The primary on-premises management console data is automatically backed up to the secondary on-premises management console every 10 minutes. The on-premises management console configurations and device data are backed up. PCAP files and logs are not included in the backup. You can back up and restore PCAPs and logs manually.

- The primary setup on the management console is duplicated on the secondary. For example, if the system settings are updated on the primary, they're also updated on the secondary.

- Before the license of the secondary expires, you should define it as the primary in order to update the license.

## About failover and failback

If a sensor can't connect to the primary on-premises management console, it automatically connects to the secondary. Your system will be supported by both the primary and secondary simultaneously, if less than half of the sensors are communicating with the secondary. The secondary takes over when more than half of the sensors are communicating with it. Failover from the primary to the secondary takes approximately three minutes. When the failover occurs, the primary on-premises management console freezes. When this happens, you can sign in to the secondary using the same sign-in credentials.

During failover, sensors continue attempts to communicate with the primary appliance. When more than half the managed sensors succeed in communicating with the primary, the primary is restored. The following message appears on the secondary console when the primary is restored:

:::image type="content" source="media/how-to-set-up-high-availability/secondary-console-message.png" alt-text="Screenshot of a message that appears at the secondary console when the primary is restored.":::

Sign back in to the primary appliance after redirection.

## Prerequisites

Before you perform the procedures in this article, verify that you've met the following prerequisites:

- Make sure that you have an [on-premises management console installed](./ot-deploy/install-software-on-premises-management-console.md) on both a primary appliance and a secondary appliance.

    - Both your primary and secondary on-premises management console appliances must be running identical hardware models and software versions.
    - You must be able to access to both the primary and secondary on-premises management consoles as a [privileged user](references-work-with-defender-for-iot-cli-commands.md), for running CLI commands. For more information, see [On-premises users and roles for OT monitoring](roles-on-premises.md).

- Make sure that the primary on-premises management console is fully [configured](how-to-manage-the-on-premises-management-console.md), including at least two [OT network sensors connected](how-to-manage-individual-sensors.md#connect-a-sensor-to-the-management-console) and visible in the console UI, and scheduled backups or VLAN settings. All settings are applied to the secondary appliance automatically after pairing.

- Make sure that your SSL/TLS certificates meet required criteria. For more information, see [Deploy OT appliance certificates](how-to-deploy-certificates.md).

- Make sure that your organizational security policy grants you access to the following services, on the primary and secondary on-premises management console. These services also allow the connection between the sensors and secondary on-premises management console:

    |Port|Service|Description|
    |----|-------|-----------|
    |**443 or TCP**|HTTPS|Grants access to the on-premises management console web console.|
    |**22 or TCP**|SSH|Syncs the data between the primary and secondary on-premises management console appliances|
    |**123 or UDP**|NTP| The on-premises management console's NTP time sync. Verify that the active and passive appliances are defined with the same timezone.|

## Create the primary and secondary pair

  > [!IMPORTANT]
> Run commands with sudo only where indicated. If not indicated, do not run with sudo.

1. Power on both the primary and secondary on-premises management console appliances.

1. **On the secondary appliance**, use the following steps to copy the connection string to your clipboard:

    1. Sign in to the secondary on-premises management console, and select **System Settings** on the left.

    1. In the **Sensor Setup - Connection String** area, under **Copy Connection String**, select the :::image type="icon" source="media/how-to-troubleshoot-the-sensor-and-on-premises-management-console/eye-icon.png" border="false"::: button to view the full connection string.

    1.  The connection string is composed of the IP address and the token. The IP address is before the colon, and the token is after the colon. Copy the IP address and token separately. For example, if your connection string is ```172.10.246.232:a2c4gv9de23f56n078a44e12gf2ce77f```, copy the IP address ```172.10.246.232``` and the token ```a2c4gv9de23f56n078a44e12gf2ce77f``` separately.

        :::image type="content" source="media/how-to-set-up-high-availability/copy-connection-string-second-part.png" alt-text="Screenshot showing to copy each part of the connection string to use in the following command." lightbox="media/how-to-set-up-high-availability/copy-connection-string-second-part.png":::

1. **On the primary appliance**, use the following steps to connect the secondary appliance to the primary via CLI:

    1. Sign in to the primary on-premises management console via SSH to access the CLI, and then run:

        ```bash
        sudo cyberx-management-trusted-hosts-add -ip <Secondary IP> -token <Secondary token>
        ```

        where `<Secondary IP>` is the IP address of the secondary appliance and `<Secondary token>` is the second part of the connection string after the colon, which you'd copied to the clipboard earlier.
        
        For example:

        ```sudo cyberx-management-trusted-hosts-add -ip 172.10.246.232 -token a2c4gv9de23f56n078a44e12gf2ce77f```

        The IP address is validated, the SSL/TLS certificate is downloaded to the primary appliance, and all sensors that are connected to the primary appliance are connected to the secondary appliance.

    1. Apply your changes on the primary appliance. Run:

        ```bash
        sudo cyberx-management-trusted-hosts-apply
        ```
    1. Verify that the certificate is installed correctly on the primary appliance. Run:

        ```bash
        cyberx-management-trusted-hosts-list
        ```

1. Allow the connection between the primary and secondary appliances' backup and restore process:

    - **On the primary appliance**, run:
    
        ```bash
        cyberx-management-deploy-ssh-key <secondary appliance IP address>
        ```
  
    - **On the secondary appliance**, sign in via SSH to access the CLI, and run:

        ```bash
        cyberx-management-deploy-ssh-key <primary appliance IP address>
        ```

1. Verify that the changes have been applied on the secondary appliance. **On the secondary appliance**, run: 
    
    ```bash
    cyberx-management-trusted-hosts-list
    ```

### Track high availability activity

The core application logs can be exported to the Defender for IoT support team to handle any high availability issues.  

**To access the core logs**:

1. Sign into the on-premises management console and select **System Settings** > **Export**. For more information on exporting logs to send to the support team, see [Export logs from the on-premises management console for troubleshooting](how-to-troubleshoot-the-sensor-and-on-premises-management-console.md#export-logs-from-the-on-premises-management-console-for-troubleshooting).

## Update the on-premises management console with high availability

To update an on-premises management console that has high availability configured, you'll need to:

1. Disconnect the high availability from both the primary and secondary appliances.
1. Update the appliances to the new version.
1. Reconfigure the high availability back onto both appliances.

Perform the update in the following order. Make sure each step is complete before you begin a new step.

**To update an on-premises management console with high availability configured**:

1. Disconnect the high availability from both the primary and secondary appliances:

    **On the primary:**
    
    1. Get the list of the currently connected appliances. Run: 

        ```bash
        cyberx-management-trusted-hosts-list
        ```

    1. Find the domain associated with the secondary appliance and copy it to your clipboard. For example:

        :::image type="content" source="media/how-to-set-up-high-availability/update-high-availability-domain.jpg" alt-text="Screenshot showing the domain associated with the secondary appliance." lightbox="media/how-to-set-up-high-availability/update-high-availability-domain.jpg":::

    1. Remove the secondary domain from the list of trusted hosts. Run:
    
        ```bash
        sudo cyberx-management-trusted-hosts-remove -d [Secondary domain]
        ```
    
    1. Verify that the certificate is installed correctly. Run:
    
        ```bash
        sudo cyberx-management-trusted-hosts-apply
        ```
    
    **On the secondary:**
    
    1. Get the list of the currently connected appliances. Run: 

        ```bash
        cyberx-management-trusted-hosts-list
        ```

    1. Find the domain associated with the primary appliance and copy it to your clipboard.

    1. Remove the primary domain from the list of trusted hosts. Run:
    
        ```bash
        sudo cyberx-management-trusted-hosts-remove -d [Primary domain]
        ```
    
    1. Verify that the certificate is installed correctly. Run:
    
        ```bash
        sudo cyberx-management-trusted-hosts-apply
        ```

1. Update both the primary and secondary appliances to the new version. For more information, see [Update an on-premises management console](update-ot-software.md#update-an-on-premises-management-console).

1. Set up high availability again, on both the primary and secondary appliances. For more information, see [Create the primary and secondary pair](#create-the-primary-and-secondary-pair).


## Next steps

For more information, see [Activate and set up your on-premises management console](how-to-activate-and-set-up-your-on-premises-management-console.md).
