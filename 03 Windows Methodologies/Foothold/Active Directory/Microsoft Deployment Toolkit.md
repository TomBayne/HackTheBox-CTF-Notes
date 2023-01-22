## MDT and PXE

Microsoft Deployment Toolkit is a service that allows organisations to automatically deploy Windows. It allows for one base image to be maintained at one centralised location and deployed automatically across the network. This can be paired with PXE network booting to create a truly 'plug and play' experience for deploying new machines. When MDT and PXE boot are used in conjunction, the process for deploying a new machine is as simple as connecting the device to the network via Ethernet and powering it on. The machine will then use TFTP to receive the boot image from the PXE server and begin installation (which can be configured to be automatic with MDT.)

This presents two opportunities for exploitation:

- The boot image can be replaced/modified to inject a Privilege Escalation vector (such as a Local Administrator account) that can be exploited after installation.
- The boot image may contain pre-configured accounts, from which the password can be scraped and used on a target machine.

## Scraping credentials

Once the IP of the MDT/PXE server has been enumerated (through DHCP in real world, through another method in CTF), enumeration of boot image files can begin. A PXE server may contain files a number of different system architectures. Once the relevant `.bcd` file has downloaded (usually via `tftp -i`), it can be examined using the [PowerPXE](https://github.com/wavestone-cdt/powerpxe) PowerShell script.

Use the `Get-WimFile` function in PowerPXE to locate the location of the actual boot image on the server, then use `tftp -i` again to download that file.

With the new file, use the command `Get-FindCredentials -WimFile preboot.wim` to scrape credentials.