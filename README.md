# terraform-hxmin-pingfederate

This repository provides resources for deploying and configuring PingFederate.

## Table of Contents

- [Installing PingFederate on Ubuntu 24.04](#installing-pingfederate-on-ubuntu-2404)
- [Integrating PingFederate with Microsoft Entra ID](#integrating-pingfederate-with-microsoft-entra-id)
- [References](#references)

## Installing PingFederate on Ubuntu 24.04

This guide walks you through the installation of PingFederate on an Ubuntu 24.04 VM.

### Prerequisites

- Ubuntu 24.04 LTS VM with root or sudo access
- At least 4GB RAM and 20GB disk space
- Java 17 or later (PingFederate requires Java)

### Step 1: Update System Packages

```bash
sudo apt update && sudo apt upgrade -y
```

### Step 2: Install Java

PingFederate requires Java 17 or later. Install OpenJDK 17:

```bash
sudo apt install openjdk-17-jdk -y
```

Verify the Java installation:

```bash
java -version
```

### Step 3: Create PingFederate User

It's a best practice to run PingFederate as a non-root user:

```bash
sudo useradd -m -s /bin/bash pingfederate
```

### Step 4: Download PingFederate

1. Visit the [Ping Identity Downloads](https://www.pingidentity.com/en/resources/downloads/pingfederate.html) page
2. Download the latest PingFederate server package (you'll need a Ping Identity account)
3. Transfer the downloaded file to your Ubuntu VM

Alternatively, if you have a direct download link (requires authentication - replace `<PINGFEDERATE_DOWNLOAD_URL>` with your actual download URL):

```bash
cd /opt
# Note: Replace <PINGFEDERATE_DOWNLOAD_URL> with the actual URL from your Ping Identity account
sudo wget <PINGFEDERATE_DOWNLOAD_URL>
```

### Step 5: Extract and Install PingFederate

```bash
cd /opt
sudo unzip pingfederate-*.zip
sudo mv pingfederate-* pingfederate
sudo chown -R pingfederate:pingfederate /opt/pingfederate
```

### Step 6: Configure PingFederate

Edit the run.properties file to configure basic settings:

```bash
sudo nano /opt/pingfederate/bin/run.properties
```

Key configurations to review:
- `pf.admin.https.port` (default: 9999)
- `pf.console.bind.address` (set to 0.0.0.0 to allow external access)
- `pf.admin.hostname` (set to your server's hostname or IP)

### Step 7: Configure Firewall

Allow traffic on PingFederate ports:

```bash
sudo ufw allow 9999/tcp  # Admin console
sudo ufw allow 9031/tcp  # Runtime engine
sudo ufw enable
```

### Step 8: Start PingFederate

Start PingFederate as the pingfederate user:

```bash
sudo su - pingfederate
cd /opt/pingfederate/bin
./run.sh
```

For production environments, configure PingFederate as a systemd service:

```bash
sudo nano /etc/systemd/system/pingfederate.service
```

Add the following content:

```ini
[Unit]
Description=PingFederate Server
After=network.target

[Service]
Type=simple
User=pingfederate
Group=pingfederate
WorkingDirectory=/opt/pingfederate/bin
ExecStart=/opt/pingfederate/bin/run.sh -c
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Enable and start the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable pingfederate
sudo systemctl start pingfederate
```

### Step 9: Access PingFederate Admin Console

1. Open a web browser and navigate to: `https://<your-server-ip>:9999/pingfederate/app`
2. Default credentials: `administrator` / `2FederateM0re`
3. **⚠️ CRITICAL SECURITY WARNING**: Change the default password **immediately** upon first login! Using default credentials in production environments poses a severe security risk and can lead to unauthorized access and system compromise.

## Integrating PingFederate with Microsoft Entra ID

This section covers the integration of PingFederate with Microsoft Entra ID (formerly Azure AD) for federated authentication.

### Prerequisites

- PingFederate installed and running
- Microsoft Entra ID tenant with administrative access
- Azure AD Premium P1 or P2 license (required for SAML-based federation)

### Architecture Overview

PingFederate will act as an Identity Provider (IdP) that federates with Entra ID, allowing users to authenticate through PingFederate before accessing Microsoft 365 and Azure resources.

### Step 1: Configure PingFederate as SAML IdP

#### 1.1 Create IdP Connection in PingFederate

1. Log in to PingFederate Admin Console (`https://<server>:9999/pingfederate/app`)
2. Navigate to **Applications** → **Integration** → **IdP Connections**
3. Click **Create New**

#### 1.2 Configure Connection Settings

1. **Connection Type**: Select "Browser SSO Profiles"
2. **Connection Template**: Select "Microsoft Azure AD"
3. **Partner Entity ID**: Enter `urn:federation:MicrosoftOnline`
4. **Connection Name**: Enter a descriptive name (e.g., "Microsoft Entra ID")

#### 1.3 Configure Browser SSO

1. Select **SAML 2.0** as the protocol
2. Configure **IdP Browser SSO**:
   - Enable "IdP-Initiated SSO"
   - Enable "SP-Initiated SSO"
3. Set **Assertion Lifetime**: 5 minutes (recommended)

#### 1.4 Configure Credentials

1. In the **Credentials** tab:
   - Select or create a signing certificate
   - Configure signature algorithms (SHA-256 recommended)

#### 1.5 Download IdP Metadata

1. Navigate to **Server Configuration** → **Administrative Functions** → **Metadata Export**
2. Export the IdP metadata XML file (you'll need this for Entra ID configuration)

### Step 2: Configure Microsoft Entra ID

#### 2.1 Register PingFederate as Enterprise Application

1. Sign in to [Microsoft Entra Admin Center](https://entra.microsoft.com/)
2. Navigate to **Identity** → **Applications** → **Enterprise Applications**
3. Click **New Application** → **Create your own application**
4. Enter a name (e.g., "PingFederate SSO") and select **Integrate any other application (Non-gallery)**
5. Click **Create**

#### 2.2 Configure SAML-Based Single Sign-On

1. In your new application, go to **Single sign-on**
2. Select **SAML** as the sign-on method
3. In **Basic SAML Configuration**:
   - **Identifier (Entity ID)**: Enter your PingFederate entity ID (e.g., `https://<pingfederate-host>:9031`)
   - **Reply URL (Assertion Consumer Service URL)**: Enter `https://<pingfederate-host>:9031/sp/ACS.saml2`
   - **Sign on URL**: Enter `https://<pingfederate-host>:9031/idp/startSSO.ping`

#### 2.3 Upload PingFederate Metadata

1. In section **3 - SAML Certificates**:
   - Click **Upload metadata file**
   - Upload the IdP metadata XML file exported from PingFederate
   - Click **Save**

#### 2.4 Configure User Attributes and Claims

1. In **Attributes & Claims** section, configure:
   - **Unique User Identifier (Name ID)**: `user.mail` or `user.userprincipalname`
   - Add additional claims as needed:
     - `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress`: `user.mail`
     - `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/givenname`: `user.givenname`
     - `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/surname`: `user.surname`

### Step 3: Configure Attribute Mapping in PingFederate

#### 3.1 Map Entra ID Attributes

1. Return to PingFederate Admin Console
2. Navigate to your Entra ID connection
3. Go to **Attribute Contract**
4. Map the following attributes:
   - `SAML_SUBJECT` → User identifier
   - `email` → Email address
   - `givenName` → First name
   - `surname` → Last name

#### 3.2 Configure Adapter Instance

1. Navigate to **IdP Configuration** → **Adapters**
2. Create or select an adapter instance (e.g., HTML Form Adapter)
3. Map adapter attributes to contract attributes

### Step 4: Assign Users in Entra ID

1. In Microsoft Entra Admin Center, navigate to your application
2. Go to **Users and groups**
3. Click **Add user/group**
4. Select users or groups to grant access
5. Click **Assign**

### Step 5: Test the Integration

#### 5.1 Test SP-Initiated SSO

1. Navigate to `https://myapps.microsoft.com`
2. Sign in with a test user account
3. Verify that you're redirected to PingFederate for authentication
4. After successful authentication, verify access to Microsoft resources

#### 5.2 Test IdP-Initiated SSO

1. Navigate to your PingFederate portal
2. Initiate SSO to Microsoft Entra ID
3. Verify successful authentication and access

### Troubleshooting

#### Common Issues

1. **Certificate Errors**
   - Ensure certificates are valid and not expired
   - Verify certificate trust chains

2. **Attribute Mapping Issues**
   - Check that required attributes are being sent
   - Verify attribute name formats match expectations

3. **Redirect URL Mismatches**
   - Ensure all URLs use the correct protocol (http/https)
   - Verify Reply URLs in Entra ID match ACS URLs in PingFederate

4. **User Not Found Errors**
   - Verify users are assigned to the application in Entra ID
   - Check that Name ID format matches user identifiers

#### Enable Debug Logging

In PingFederate:
```bash
cd /opt/pingfederate/bin
./run.sh --mode DEBUG
```

Check logs:
```bash
tail -f /opt/pingfederate/log/server.log
```

### Security Best Practices

1. **Use HTTPS**: Always use TLS/SSL certificates for production environments
2. **Strong Authentication**: Enable multi-factor authentication (MFA) in Entra ID
3. **Certificate Management**: Rotate certificates regularly and monitor expiration dates
4. **Audit Logging**: Enable comprehensive logging in both PingFederate and Entra ID
5. **Least Privilege**: Assign only necessary permissions to users and service accounts
6. **Regular Updates**: Keep PingFederate and all components up to date with security patches

## References

### Official Documentation

- [PingFederate Documentation](https://docs.pingidentity.com/bundle/pingfederate-latest/page/pingfederate-landing.html)
- [PingFederate Installation Guide](https://docs.pingidentity.com/bundle/pingfederate-latest/page/concepts/installPingFederateOverview.html)
- [Microsoft Entra ID SAML Documentation](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/add-application-portal-setup-sso)
- [Configure SAML-based SSO for Entra ID](https://learn.microsoft.com/en-us/entra/identity/enterprise-apps/add-application-portal-setup-sso)

### Additional Resources

- [Ping Identity Community](https://support.pingidentity.com/s/)
- [PingFederate Azure AD Integration Guide](https://docs.pingidentity.com/bundle/integrations/page/csh1563994771114.html)
- [Microsoft Entra ID Federation Documentation](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/whatis-fed)

### Support

- PingFederate Support: https://support.pingidentity.com
- Microsoft Entra ID Support: https://learn.microsoft.com/en-us/entra/fundamentals/how-to-get-support
