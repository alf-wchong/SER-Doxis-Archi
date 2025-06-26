# Steps to Achieve Complete SSO in Doxis webCube with Existing SAML

If webCube deployment already has working SAML authentication, to achieve SSO where a user logged into Windows can access webCube without any browser interaction, Kerberos authentication alongside existing SAML configuration is required.

When implementing Single Sign-On (SSO) with Kerberos in a Doxis webCube environment that uses HTTPS instead of HTTP, some special considerations must be made.

## Here's the complete step-by-step process:

### Step 1: Configure Kerberos Authentication in Server Profile

1.  Login to the Doxis webCube administration console.
2.  Navigate to `Configuration > Server profiles`.
3.  Select your existing server profile.
4.  Under "Authentication methods," check the `Kerberos` box while keeping any existing authentication methods (like SAML) enabled.
5.  Save and apply the changes.

### Step 2: Create a Network User for the Application Server

1.  On your domain controller, run Active Directory Users and Computers (`dsa.msc`).
2.  Create a dedicated service account:
    * Right-click your `domain` > `New` > `User`
    * First name: `WebCubeServer`
    * User logon name: `WebCubeServer`
    * Set a secure password and check "Password never expires".
3.  Click `Finish` to create the account.

### Step 3: Register the Network User as a Service Principal

1.  On your domain controller, open Command Prompt as Administrator.
2.  Use the `ktpass` command to create a service principal:

    `ktpass -princ HTTP/your-webcube-server@YOUR-DOMAIN.LOCAL -pType KRB5_NT_PRINCIPAL -crypto All -mapuser WebCubeServer@YOUR-DOMAIN.LOCAL -pass YourPassword -out C:\webcube-server.keytab`

    **IMPORTANT**: Note that even though your site uses HTTPS, you must still use `HTTP/` in the service principal name. Kerberos authentication uses this service type regardless of the protocol used for browser access.

3.  Verify the service principal was created:

    `setspn -L WebCubeServer`

4.  If needed, register an additional SPN for FQDN access:

    `ktpass -princ HTTP/your-webcube-server.full.domain.name@YOUR-DOMAIN.LOCAL -pType KRB5_NT_PRINCIPAL -crypto All -mapuser WebCubeServer@YOUR-DOMAIN.LOCAL -pass YourPassword -in C:\webcube-server.keytab -out C:\webcube-server.keytab`

### Step 4: Configure the Service Principal for Delegation

1.  In Active Directory Users and Computers, open properties of the WebCubeServer user.
2.  Go to the `Delegation` tab.
3.  Select `Trust this user for delegation to any service (Kerberos only)`.
4.  Click `OK` to save the changes.

### Step 5: Configure Application Server for Kerberos

1.  Copy the generated keytab file (`webcube-server.keytab`) to your application server.
2.  Configure your application server's JVM options by adding:

    `-Djava.security.krb5.conf=/path/to/krb5.conf -Djavax.security.auth.useSubjectCredsOnly=false -Djava.security.auth.login.config=/path/to/login.conf`

3.  Create or modify the `krb5.conf` file with your domain settings:

    ```
    [libdefaults]
        default_realm = YOUR-DOMAIN.LOCAL
        default_tkt_enctypes = aes256-cts-hmac-sha1-96 aes128-cts-hmac-sha1-96 rc4-hmac
        default_tgs_enctypes = aes256-cts-hmac-sha1-96 aes128-cts-hmac-sha1-96 rc4-hmac
        permitted_enctypes = aes256-cts-hmac-sha1-96 aes128-cts-hmac-sha1-96 rc4-hmac
    [realms]
        YOUR-DOMAIN.LOCAL = {
          kdc = your-domain-controller.your-domain.local
          admin_server = your-domain-controller.your-domain.local
          default_domain = your-domain.local
        }
    [domain_realm]
        .your-domain.local = YOUR-DOMAIN.LOCAL
        your-domain.local = YOUR-DOMAIN.LOCAL
    ```

4.  Create a `login.conf` file:

    ```
    com.sun.security.jgss.initiate {
        com.sun.security.auth.module.Krb5LoginModule required
        useKeyTab=true
        keyTab="/path/to/webcube-server.keytab"
        principal="HTTP/your-webcube-server@YOUR-DOMAIN.LOCAL"
        storeKey=true
        doNotPrompt=true ;
    };
    ```

5.  Restart your application server.

### Step 6: Configure Browser Settings for Kerberos

#### For Internet Explorer/Edge:

1.  Go to Internet Options > Security.
2.  Add your HTTPS webCube server URL to the Local Intranet zone.
3.  Go to Advanced tab and ensure "Enable Integrated Windows Authentication" is checked.

#### For Chrome:

1.  Set Chrome to use Windows authentication for your webCube site:
    * Create a shortcut with the parameter: `--auth-server-whitelist="*.your-domain.local"`
    * Or configure via Group Policy

#### For Firefox:

1.  Navigate to `about:config`.
2.  Set `network.negotiate-auth.trusted-uris` to include your HTTPS webCube URL.

### Step 7: Test Complete SSO

1.  Close all browser instances.
2.  Open a new browser window and navigate to your webCube URL with the Kerberos parameter:

    `https://your-webcube-server:port/webcube/?kerberoslogin=1&server=your-server-profile&system=your-organization`

3.  You should be automatically logged in using your Windows credentials.

## Troubleshooting

* Check the `webcube.log` file for any Kerberos-related errors.
* Verify time synchronization between all servers (time skew can break Kerberos).
* Ensure all SPNs are registered correctly with `setspn -L WebCubeServer`.
* Test connectivity between application server and domain controller.
* Verify that the keytab file contains the correct encryption types with `klist -kte webcube-server.keytab`.
