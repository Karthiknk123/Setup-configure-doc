# Purchasing Domain and Installing a Wildcard SSL (Namecheap + SeekaHost)

This guide shows:
- How to buy a domain on SeekaHost.
- How to purchase a PositiveSSL Wildcard on Namecheap.
- How to generate CSR and private key on your server.
- How to validate via DNS (CNAME) in SeekaHost and issue the certificate.
- How to download and install the certificate once issued.

Replace placeholders (domain names, paths, CN, organization fields, etc.) with your real values.

---

## 1) Purchase domain on SeekaHost
1. Log in to SeekaHost.
2. Go to: Domains → My Domains.
3. Click Add New.
4. Choose the domain extension and enter the desired domain name.
5. Proceed to payment to complete purchase.
6. After purchase you can manage DNS from: Dashboard → Domains → Manage your domain → DNS Management.

---

## 2) Purchase PositiveSSL Wildcard on Namecheap
1. Log in to your Namecheap account.
2. Go to Dashboard → SSL Certificates.
3. If you have no SSLs yet, click the purchase link or choose an SSL product.
4. Select the PositiveSSL Wildcard (or another wildcard SSL) and choose term (1–5 years).
5. Confirm order and complete payment.
6. Keep the order/receipt for records.

Note: Wildcard SSL covers all first-level subdomains, e.g. `*.yourdomain.com`.

---

## 3) Generate CSR and private key on your server
Generate the CSR (Certificate Signing Request) and the private key on the server you will install the certificate on. Generate once and keep the private key safe — do NOT share the private key.

1. Create a CSR config file (example at `/root/SSL_csr.cnf`):

```ini
[ req ]
default_bits       = 2048
prompt             = no
default_md         = sha256
distinguished_name = dn
req_extensions     = req_ext

[ dn ]
C  = IN
ST = Karnataka
L  = Bangalore
O  = KPTCL
OU = IT Department
CN = *.kptcl.net

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = *.kptcl.net
DNS.2 = apicms.kptcl.net
DNS.3 = cmss.kptcl.net
DNS.4 = preportcms.kptcl.net
DNS.5 = reportcms.kptcl.net
```

2. Run openssl to generate CSR and private key:

```bash
# run from directory where you want the files (e.g. /root)
openssl req -new -nodes -out wildcard.csr -newkey rsa:2048 -keyout wildcard.key -config /root/SSL_csr.cnf
```

This creates:
- `wildcard.csr` — the CSR you will paste into Namecheap.
- `wildcard.key` — the private key (keep secure, permissions 600).

Secure the key:
```bash
chmod 600 /root/wildcard.key
chown root:root /root/wildcard.key
```

You can view the CSR (to copy/paste into Namecheap):
```bash
cat /root/wildcard.csr
```

Do NOT reveal the private key to anyone:
```bash
# never run this publicly; for your reference only
cat /root/wildcard.key
```

---

## 4) Activate SSL in Namecheap (submit CSR)
1. In Namecheap Dashboard → SSL Certificates → click your purchased certificate → Details.
2. When prompted for a CSR, open `wildcard.csr` and paste its full contents into the CSR field.
3. Continue to the Domain Control Validation (DCV) step and choose validation method:
   - Email (less convenient) or
   - DNS CNAME (recommended for wildcard).

---

## 5) DNS CNAME validation (recommended)
If you chose DNS CNAME validation, Namecheap will show Host and Target values required for DCV.

1. In Namecheap Dashboard → SSL Certificates → your cert → “Get Record Details” (or similar) — copy Host and Target.
   - Host will typically be a short token (do not include the domain suffix).
   - Target will often be a long target hostname (e.g. `xxxx.comodoca.com`).

2. Log in to SeekaHost in a new tab:
   - Dashboard → Domains → Manage your domain → DNS Management → Add Record.

3. Add a new DNS record:
   - Type: CNAME
   - Host: paste Host value (only the host/token part)
   - Address / Target: paste Target value exactly
   - TTL: leave default or set low (e.g. 300) to speed propagation
   - Save changes

4. Wait for DNS propagation (usually minutes, sometimes hours). You can check propagation with:
```bash
# check CNAME (replace host and yourdomain)
dig +short host.yourdomain.com CNAME
# or
nslookup -type=CNAME host.yourdomain.com
```

Note: Do not add extra dots or spaces when entering Host/Target. Ensure the Host field is not the full FQDN (SeekaHost UI often expects the token/leftmost label only).

---

## 6) Retry validation if pending
If Namecheap still shows "Pending" after propagation:
1. In Namecheap Dashboard → Domain List (or SSL list) find your certificate.
2. Click Edit Methods or DCV method.
3. Ensure DCV is set to DNS-Based (CNAME).
4. Use the "Retry Alt DCV" or "Re-check" button to force Namecheap to re-validate the DNS record.

If it still fails:
- Re-check the CNAME values (host/target) and ensure no typos.
- Confirm the CNAME is visible publicly via `dig`/`nslookup`.
- Ensure no conflicting CNAME/A records exist for the same host.

---

## 7) Certificate issuance and download
Once Namecheap validates DCV, the certificate status will change from Pending → Issued.

1. In Namecheap Dashboard → SSL Certificates → click the issued certificate.
2. Download the certificate package. Namecheap generally provides:
   - The site certificate (CRT)
   - A CA bundle / intermediate certificate chain
   - Sometimes a combined bundle file

Keep:
- `your_domain.crt` (site certificate)
- `your_domain.ca-bundle` (intermediate bundle)
- Your previously generated `wildcard.key` (private key) — do not overwrite

---

## 8) Install SSL on your web server (NGINX example)
Place these files on your server (example location `/etc/ssl/<yourdomain>/`):
- `wildcard.key` (private key)
- `wildcard.crt` (site certificate)
- `ca-bundle.crt` (intermediate chain)

Create certificate chain file (optional, many providers include this):
```bash
cat wildcard.crt ca-bundle.crt > fullchain.pem
```

Example NGINX server block using fullchain and key:
```nginx
server {
  listen 443 ssl;
  server_name example.yourdomain.com;

  ssl_certificate /etc/ssl/yourdomain/fullchain.pem;
  ssl_certificate_key /etc/ssl/yourdomain/wildcard.key;

  # Recommended security settings (examples)
  ssl_protocols TLSv1.2 TLSv1.3;
  ssl_ciphers HIGH:!aNULL:!MD5;
  ssl_prefer_server_ciphers on;

  # Rest of configuration...
}
```

Reload NGINX:
```bash
sudo nginx -t
sudo systemctl reload nginx
```

For Apache, use the equivalent `SSLCertificateFile`/`SSLCertificateKeyFile` and `SSLCertificateChainFile` directives.

---

## 9) Post-install checks
- Verify certificate chain and expiry:
```bash
openssl s_client -connect example.yourdomain.com:443 -servername example.yourdomain.com -showcerts
```
- Check certificate details:
```bash
openssl x509 -in /etc/ssl/yourdomain/fullchain.pem -noout -text
```
- Use external SSL testers (e.g., SSL Labs) to verify chain and configuration.

---

## 10) Common issues & troubleshooting
- DNS propagation: wait and re-check with `dig` before retrying Namecheap validation.
- Wrong Host value: SeekaHost UI often expects only the token part (not the full domain).
- Conflicting records: ensure there is no A or other CNAME record for the same host entry.
- Private key mismatch: the `.csr` was generated from `wildcard.key`. If a different private key was used, the certificate won't match.
- Web server errors after install: check server logs and run `nginx -t` or `apachectl configtest`.

---

## 11) Security reminders
- Never share your private key (`wildcard.key`).
- Use strong file permissions (600) for private key.
- Back up your private key and certificate bundle securely.
- For production, consider automating renewals or reminders (wildcard certs still require renewal—Namecheap can auto-renew if configured).

---

If you want, I can:
- Produce the exact `/root/SSL_csr.cnf` content tailored to a different domain or organization values.
- Produce Nginx and Apache example configs with more security hardening (HSTS, OCSP stapling).
- Walk through the SeekaHost UI steps if you provide a screenshot or the specific SeekaHost DNS form fields you see.
