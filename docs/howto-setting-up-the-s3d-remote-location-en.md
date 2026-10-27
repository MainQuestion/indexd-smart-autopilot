# Setting Up an IndexD Remote Location on Windows: From Local Startup to HTTPS

A practical report on setting up an IndexD remote location on Windows.
It covers the steps followed in the working environment: IndexD
configuration, domain setup, Caddy, TLS certificate issuance, reverse
proxying, S3D authorization, and starting the location.

> **Scope and accuracy.** This guide distinguishes confirmed actions and
> results from details that could not be fully recovered. The final
> `Caddyfile` and some intermediate commands are not guessed. Never
> publish DNS-provider or ZeroSSL secrets.

## 1. System layout

The following addresses and ports were used:

  ----------------------------------------------------------------------------------------
  Component               Address / port                           Purpose
  ----------------------- ---------------------------------------- -----------------------
  Public hostname         `mainask.remotewire.net`                 Remote location domain

  Public HTTPS endpoint   `https://mainask.remotewire.net:18443`   Address used for
                                                                   external connections

  Caddy                   HTTPS on port `18443`                    TLS termination and
                                                                   reverse proxy

  IndexD application API  `127.0.0.1:19982`                        Local upstream for
                                                                   Caddy

  IndexD syncer           `127.0.0.1:19981`                        Local syncer

  IndexD admin API        `0.0.0.0:19980`                          Admin API bound to all
                                                                   interfaces

  PostgreSQL              `127.0.0.1:5432`                         Local database

  Data directory          `D:\IndexD\data\`                        IndexD data

  S3D executable          `C:\S3D\s3d.exe`                         Remote-location process
  ----------------------------------------------------------------------------------------

These values belong to one specific installation. Verify them before
reusing them. In particular, an admin API bound to `0.0.0.0:19980`
listens on all network interfaces; restrict access according to your
installation's security requirements.

## 2. Check local IndexD before configuring HTTPS

The local application endpoint was:

``` text
127.0.0.1:19982
```

To check its root URL:

``` powershell
curl.exe -i http://127.0.0.1:19982/
```

**Observed result:** IndexD returned `404` for `/`. This alone does not
mean the service is broken: the root path may not have a handler. Later,
a `404` response from the public HTTPS endpoint also confirmed that the
request had passed through Caddy and reached IndexD.

Distinguish between: - a connection refusal or timeout, which may
indicate an availability, routing, port, or process issue; - an HTTP
`404` from IndexD, which means the connection succeeded but the specific
`/` path was not found.

## 3. IndexD configuration

The following values were recorded during setup:

``` yaml
applicationAPI: 127.0.0.1:19982
adminAPI: 0.0.0.0:19980
syncer: 127.0.0.1:19981
advertiseURL: https://mainask.remotewire.net:18443
data: d:\IndexD\data\
bootstrap: true
network: mainnet
```

This is a list of values observed during the work, not necessarily a
complete configuration file. Field names may vary by application
version. Preserve other required settings and verify the syntax against
the configuration for your version.

Initially, `advertiseURL` pointed to a local address:

``` yaml
advertiseURL: http://localhost:19982
```

After the public HTTPS endpoint was prepared, it was changed to:

``` yaml
advertiseURL: https://mainask.remotewire.net:18443
```

This advertises the externally reachable address to other participants
rather than the computer's internal loopback address.

## 4. Domain and DNS

The public endpoint used this hostname:

``` text
mainask.remotewire.net
```

DNS was managed through Dynu. The domain must resolve to the external
address of the machine receiving connections, and Caddy must be able to
prove control of the hostname when issuing the certificate.

Before reproducing the setup, verify that: 1. the hostname resolves to
the current public IP address; 2. inbound traffic to the selected public
port is allowed by the router and/or Windows Firewall; 3. Caddy listens
on the port specified in the public URL; 4. if using a DNS challenge,
Caddy has the relevant DNS-provider module and valid Dynu API
credentials.

The exact DNS records, router rules, and Windows Firewall rules were not
preserved in the available history, so this guide does not present them
as commands that were definitely executed.

## 5. Configure Caddy as an HTTPS reverse proxy

Caddy was used as an HTTPS reverse proxy: it accepted requests on the
public address and forwarded them to the local IndexD application API.

Public endpoint:

``` text
https://mainask.remotewire.net:18443
```

Local upstream:

``` text
127.0.0.1:19982
```

A schematic proxy configuration looks like this:

``` caddyfile
https://mainask.remotewire.net:18443 {
    # TLS/issuer and DNS challenge settings must match
    # your Caddy build and Dynu configuration.
    reverse_proxy 127.0.0.1:19982
}
```

This is a **schematic fragment**, not the complete final `Caddyfile`:
the full TLS and DNS-provider configuration was not preserved in the
available history. Do not replace your working file with this fragment
alone.

During initial testing, Caddy returned a test response:

``` caddyfile
respond "indexD HTTPS endpoint is ready" 200
```

This allowed HTTPS and the route to Caddy to be tested separately from
IndexD. The test `respond` directive was later replaced with
`reverse_proxy 127.0.0.1:19982`.

## 6. Challenge #1: Let's Encrypt rate limits

The first attempt to issue a certificate through the Let's Encrypt
production ACME endpoint returned HTTP `429`.

The response said that **50 certificates had already been issued for the
registered domain `remotewire.net` within 168 hours**. It specified this
retry time:

``` text
2026-10-26 18:39:13 UTC
```

This was a certificate-issuance rate limit, not a reverse-proxy error
and not evidence of an incorrect IndexD configuration.

The Let's Encrypt staging ACME environment was used to test the initial
setup:

``` text
https://acme-staging-v02.api.letsencrypt.org/directory
```

Staging is intended for testing and does not issue the normal
browser-trusted production certificate. Therefore, a successful staging
test did not complete the task of obtaining a public certificate.

## 7. Challenge #2: switching to ZeroSSL

ZeroSSL was selected for the next attempt, using this ACME endpoint:

``` text
https://acme.zerossl.com/v2/DV90
```

ZeroSSL ACME required External Account Binding (EAB). The account
provided: - API Key; - EAB KID; - EAB HMAC Key.

These values are used when registering the ACME account. **Do not put
real values in public documentation, screenshots, repositories, or
logs.** Supply them through a secure method supported by your particular
Caddy configuration.

Caddy `v2.11.7` logs confirmed that the ZeroSSL ACME account was
registered with EAB. Caddy then started creating a certificate order
(`newOrder`), but one request timed out while waiting for HTTP/2
response headers after approximately 30 seconds, and Caddy retried.

That intermediate log entry alone did not prove either final failure or
successful issuance. The outcome had to be checked in subsequent logs
and by testing the actual HTTPS connection.

## 8. Verify HTTPS and the reverse proxy

The first test checked Caddy's own response:

``` text
indexD HTTPS endpoint is ready
```

During troubleshooting, `curl.exe` was also run with `-k`, for example:

``` powershell
curl.exe -k -i https://mainask.remotewire.net:18443/
```

The `-k` option disables certificate trust validation. It can be useful
for diagnosing a TLS connection, but it is **not suitable for the final
validation of a trusted certificate**.

After Caddy was switched to reverse proxying, this request:

``` powershell
curl.exe -i https://mainask.remotewire.net:18443/
```

returned HTTP `404` from IndexD. The ZeroSSL certificate was ultimately
checked and found valid and trusted, and requests to the public endpoint
reached IndexD.

Why `404` was acceptable in this case: - the TLS connection was
established; - the request passed through Caddy to the upstream; -
IndexD replied that no handler existed for `/`.

For the final certificate check, run the request **without** `-k`:

``` powershell
curl.exe -i https://mainask.remotewire.net:18443/
```

If certificate validation fails, do not keep using `-k` as a workaround.
Check the certificate hostname, expiry, trust chain, system clock, and
Caddy logs.

## 9. Authorize S3D and start the remote location

S3D was prepared in this directory:

``` text
C:\S3D
```

### 9.1. Sign in

In PowerShell:

``` powershell
Set-Location C:\S3D
.\s3d.exe login
```

The program provided an authorization URL. It was opened in a browser,
then a new connection key was created in the IndexD interface and
authorization was approved.

A successful sign-in displayed:

``` text
Connection approved
```

and PowerShell reported:

``` text
Login successful. You can now start s3d
```

If authorization does not complete, check the URL opened in the browser,
the selected key, and the console output. Do not publish keys or tokens.

### 9.2. S3D configuration

The configuration was found in the user profile directory:

``` text
C:\Users\admin\AppData\Roaming\s3d\
```

It was copied to:

``` text
C:\S3D
```

The data distribution settings in the configuration were changed to:

``` yaml
dataShards: 3
parityShards: 2
```

These are values from this particular installation, not universal
recommendations. Check what the settings mean before changing them, and
preserve the other configuration fields.

### 9.3. Start S3D

After successful authorization:

``` powershell
Set-Location C:\S3D
.\s3d.exe
```

Make sure the process remains active, the console shows no errors, and
the location's status appears in the IndexD interface. Starting the
process alone does not prove that all external connections and
operations are working correctly.

## 10. Troubleshooting timeline

  -------------------------------------------------------------------------------
  Stage                   Observation                     Conclusion
  ----------------------- ------------------------------- -----------------------
  Local IndexD check      `http://127.0.0.1:19982/`       The root URL had no
                          returned `404`                  handler; this alone did
                                                          not indicate service
                                                          failure

  First certificate       Let's Encrypt returned HTTP     The rate limit had to
  attempt                 `429` because of the            be respected instead of
                          50-certificates-per-168-hours   repeatedly requesting
                          limit for `remotewire.net`      issuance

  ACME test environment   Let's Encrypt staging endpoint  Suitable for testing,
                          was used                        but not a final trusted
                                                          certificate

  Switch to ZeroSSL       EAB KID and HMAC Key were       ACME account
                          required                        credentials were needed

  ZeroSSL registration    Caddy `v2.11.7` registered the  Registration succeeded;
                          ACME account                    that did not itself
                                                          mean the certificate
                                                          was issued

  Certificate order       Timeout waiting for HTTP/2      Subsequent events had
                          headers during `newOrder`,      to be checked; one log
                          followed by a retry             entry was not
                                                          conclusive

  HTTPS through Caddy     Caddy's test response returned  HTTPS to Caddy itself
                          `200`                           was confirmed

  Reverse proxy           Public request returned `404`   The request reached the
                          from IndexD                     upstream; `/` had no
                                                          handler

  S3D authorization       `Connection approved` and       Remote-location
                          `Login successful`              authorization completed

  S3D startup             `.\s3d.exe` was started from    The process started;
                          `C:\S3D`                        readiness must be
                                                          verified separately
  -------------------------------------------------------------------------------

## 11. Final architecture

``` text
External client
      |
      | HTTPS: mainask.remotewire.net:18443
      v
    Caddy
      |
      | reverse_proxy
      v
IndexD application API
127.0.0.1:19982

S3D on Windows
C:\S3D\s3d.exe
      |
      | browser authorization and IndexD key
      v
IndexD remote location
```

This diagram shows the role of each component; it does not replace
verification of the actual network rules and configuration files.

## 12. Checklist

-   [ ] IndexD is running and the local application endpoint is known.
-   [ ] The local endpoint response has been checked.
-   [ ] `mainask.remotewire.net` resolves to the correct public address.
-   [ ] Caddy listens on public HTTPS port `18443`.
-   [ ] Caddy proxies requests to `127.0.0.1:19982`.
-   [ ] ACME settings match the chosen certificate authority.
-   [ ] If using ZeroSSL ACME, EAB credentials are configured securely.
-   [ ] The certificate validates without `curl -k`.
-   [ ] `advertiseURL` is set to `https://mainask.remotewire.net:18443`.
-   [ ] `.\s3d.exe login` has been completed.
-   [ ] `Connection approved` and `Login successful` were displayed.
-   [ ] The S3D configuration is in the expected directory.
-   [ ] `.\s3d.exe` is running, errors have been checked, and the
    location's status has been verified in IndexD.


