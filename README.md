# Better-AGW-ACME

A very simple way to do ACME in Azure, using Let's Encrypt.

# Features:

- Strong keysizes for issued certificates (RSA 4096, SECP384r1)
- HTTP-01 challenges
- Azure Application Gateway aware
- Exposes API for monitoring certificate issuance
- Configures Let's Encrypt to email you with alerts regarding issuance
- Forces "Must Staple" OCSP attribute on certificates, to improve end user privacy
- Supports RFC 8657 accounturi pinning, preventing malicious actors from misissuing certificates
- Automatically renews certificates after 10% of their lifetime has passed, which is ~9 days currently
- Automatically pushes certificates to Azure KeyVault
- Automatically fetches account secrets from Azure KeyVault
- Automatically setups up account secret in Azure KeyVault if one has not been set
- Requires minimal security patching - uses Azure App Service provided userspace, and automatically patches userspace
- Runs in Azure App Service, without needing a 3rd party container image
- No 3rd party packages required, uses only packages in distro repos
- Off the shelf components - uses Apache mod_md as an ACME client
- Minimal maintainence overall

# App Service Configuration:

- You need to setup Azure App Service to use the ".NET 8.0" runtime version. We don't actually use .NET, we just care about the userspace provided by this container.
- You need to setup the Azure App Service startup command to be "bash startup.bash".
- You need to ensure that the Azure App Service is having /.well-known/acme-challenge/ reverse proxied by Azure Application Gateway.
- You optionally, if you wish to view certificate issuing actions, ensure that the Azure App Service is having /.httpd/ reverse proxied by Azure Application Gateway.
- You need to ensure that the Azure App Service has access to the following servers:

  - The Debian package repos, https://security.debian.org TLS port 443
  - The Let's Encrypt ACME API, https://acme-v02.api.letsencrypt.org TLS port 443
  - The Azure metadata API, 169.254.169.254 on HTTP port 80 (internal)

- You need to configure the below 2 environment variables with your desired values. 

  Do not include a training "/" on the value in KEYVAULT_HOSTNAME_FOR_STORAGE.

  ```CONTACT_EMAIL="yourcontactemail@example.invalid"```

  ```KEYVAULT_HOSTNAME_FOR_STORAGE="https://your-keyvault.kv.azure.net"```

# DNS configuration:

In order to significantly improve overall ACME assurance, you must configure a CAA record for your DNS zone. This is a "set it and forget it task". One of very few such tasks in the IT industry.

For example, for my hostname I wish to issue certificates for, thisis.ademo.webpage, I could assign a CAA record to either ademo.webpage, which would impact the entire apex DNS zone and all children, or I could just do it for thisis.ademo.webpage.

Here is what the DNS record would look like:

```thisis.ademo.webpage.         3600    IN      CAA     0 issue "letsencrypt.org; validationmethods=http-01; accounturi=https://acme-v02.api.letsencrypt.org/acme/acct/YOUR_ACCOUNT_ID_FROM_KEYVAULT"```

Note, you would replace the contents of YOUR_ACCOUNT_ID_FROM_KEYVAULT in the above DNS record, with that of what is stored in Azure KeyVault, as part of ACME account credentials.

This will do 2 things:

- Prevent other CAs other than Let's Encrypt from issuing TLS certificates for this DNS zone
- Prevent malicious actors from abusing Let's Encrypt using techniques such as BGP and DNS zone hijacking, from issuing certificates, as you have a CAA record pinning the account ID, and the credentials for this account are in your Azure KeyVault

It is strongly reccomended to also deploy DNSSEC, as this will further improve resilience of DNS queries performed by the CA when doing ACME challenges, irrespective of challenge type.


# Ideas

- Consider potentially implementing this using Azure Automation and Azure Static Websites served from Blob Storage instead?
