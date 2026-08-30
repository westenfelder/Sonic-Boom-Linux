# Sonic-Boom-Linux
Microsoft AVD with CAC passthrough on Linux  

## Background
As of August 2026, Microsoft does not provide an official Linux client to connect to Azure Virtual Desktop (AVD). Linux users must use the AVD web client, which does [not support](https://usaf.dps.mil/sites/SBVP/SitePages/AVD---FAQs.aspx) Common Access Card (CAC) passthrough. Lack of CAC passthrough prevents authenticating to websites and signing documents. This repo provides instructions to resolve these issues.

CAC passthrough is officially supported using the [Windows App](https://apps.microsoft.com/detail/9n1f85v9t8bn?hl=en-US&gl=US) on Windows operating systems.

## Notes
- The Air Force's Enterprise IT as a Service Virtual Desktop Infrastructure (EITaaS VDI) replaced Sonic Boom in April 2026. These instructions support the EITaaS VDI and may work for other remote desktops powered by AVD.
- Running the official Windows App inside a virtual machine or container (such as [winboat](https://github.com/winboat-org/winboat)) may be a simpler solution for some users. These instructions avoid the overhead of virtualization.
- This repo only provides manual connection instructions and has only been tested on Ubuntu 24.04 x86_64. For a more mature project, please see [EITaaS-Linux](https://github.com/sjtrotter/EITaaS-Linux).

## Instructions
1. Visit [https://rdweb.wvd.azure.us/arm/webclient](https://rdweb.wvd.azure.us/arm/webclient)
    - Click the settings cog and select "Download the rdp file"
    - Click "Desktop" to download the `Desktop.rdpw` file

2. Install dependencies

    ```bash
    sudo apt update && sudo apt install -y freerdp3-x11 pcscd pcsc-tools libccid opensc libpcsclite1
    ```

3. Check smart card reader and CAC
    - Ensure the following commands run without root permissions (do NOT use sudo)

    ```bash
    pcsc_scan -c
    pkcs11-tool --test --login
    ```

    - If either command fails, you may need to modify pcsc permissions

    ```bash
    sudo mkdir -p /etc/polkit-1/localauthority/50-local.d/
    sudo mkdir -p /etc/polkit-1/rules.d/
    
    sudo tee /etc/polkit-1/localauthority/50-local.d/99-pcscd.pkla > /dev/null <<EOF
    [Allow pcscd access for all users]
    Identity=unix-user:*
    Action=org.debian.pcsc-lite.access_pcsc;org.debian.pcsc-lite.access_card;org.pcsc-lite.access_pcsc;org.pcsc-lite.access_card
    ResultAny=yes
    ResultInactive=yes
    ResultActive=yes
    EOF
    
    sudo tee /etc/polkit-1/rules.d/99-pcscd.rules > /dev/null <<EOF
    polkit.addRule(function(action, subject) {
        if (action.id.indexOf("pcsc-lite.access_pcsc") > -1 || action.id.indexOf("pcsc-lite.access_card") > -1) {
            return polkit.Result.YES;
        }
    });
    EOF
    
    sudo systemctl restart polkit
    sudo systemctl restart pcscd
    ```

4. Launch connection

    ```bash
    xfreerdp3 Desktop.rdpw \
        /gateway:type:arm \
        /sec:aad \
        /azure:ad:login.microsoftonline.us,tenantid:common,avd-access:https://login.microsoftonline.com/common/oauth2/nativeclient \
        /cert:ignore \
        /smartcard
    ```

5. Authenticate and capture the OAuth authorization code
    - Copy the authentication link(s) from xfreerdp3
    - Open Dev Tools in browser -> Network Tab -> Click "Keep log" -> Start recording
    - Paste the link and authenticate
    - Find "nativeclient?code=..." -> Right click -> Copy -> Copy URL
    - Paste the redirection URL back into xfreerdp3
    - The connection may require multiple authentications

6. Test CAC passthrough
   - In the RDP session, attempt CAC authentication on a website (such as [LeaveWeb](https://leave.af.mil/login/1))

