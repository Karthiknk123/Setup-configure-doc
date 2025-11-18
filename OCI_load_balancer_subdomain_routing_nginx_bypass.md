# Subdomain Loading (bypassing NGINX) using OCI Load Balancer

This guide shows how to configure an Oracle Cloud Infrastructure (OCI) Load Balancer to route traffic for multiple subdomains directly to backend services (so the subdomain loads without an extra NGINX reverse proxy). The approach uses backend sets, hostnames, and routing policies on the OCI Load Balancer.

Summary
- Create backend sets for your services
- Add backend instances (or instance pools) and set the service port
- Create hostnames on the Load Balancer that represent the subdomains
- Create routing policies that match the Host header (subdomain) and route to the correct backend set
- Attach routing policies to the HTTPS listener
- Add DNS A records pointing the subdomain to the Load Balancer IP

---

## Preconditions
- You have an OCI Load Balancer (public or private) provisioned.
- DNS zone (at your DNS provider) where you can create A records for subdomains.
- Backend instances/services are reachable from the Load Balancer and listening on the intended ports.
- SSL termination will typically happen at the Load Balancer (HTTPS listener). If you want end-to-end TLS, configure backend SSL or use TCP listeners.

---

## 1) Map subdomain to A record (DNS)
- In your DNS provider (or OCI DNS if used), create an A record for each subdomain:
  - Host / Name: subdomain (for example `api` for `api.example.com`)
  - Type: A
  - Value: the public IP address of the Load Balancer
  - TTL: default or low (300) while testing
- Wait for DNS propagation (verify with `dig` or `nslookup`).

---

## 2) Create Backend Set(s)
1. In the OCI console, open your Load Balancer and go to **Backend Sets** → **Create Backend Set**.
2. Provide:
   - Name: e.g., `backend-api`, `backend-web`
   - Policy: `Weighted round robin` (or a policy of your choice)
   - Health check: set Protocol = TCP and Port = the service port your backend exposes (e.g., 8080)
   - Leave other settings default unless you require SSL, session persistence, or custom timeouts.
3. Create the backend set.

4. After creating the backend set, click it and go to **Backends** → **Add Backend**:
   - Add each backend instance or IP and set the backend port to the service port (for example 5678).
   - Save changes.

5. Update health check from the backend set above choose **Actions** → **Update Health Check**, set Protocol = TCP and Port = running service port.

Notes:
- If your backends run on different ports, create separate backend sets per service/port.
- Health checks using TCP are simple and effective if you only need port reachability; use HTTP(S) if you want application-level health checks.

---

## 3) Create Hostname(s) on Load Balancer
1. In Load Balancer → Hostnames → Create Hostname
2. Provide:
   - Name: a friendly name (e.g., `hostname-api`)
   - Hostname: the subdomain (e.g., `api.example.com`) — do not include `http://` or `https://`
3. Save.

You can create multiple hostnames for different subdomains you plan to route.

---

## 4) Create Routing Policies (Host-based routing)
Routing policies decide which backend set receives traffic based on request attributes (header, path, etc.). We will match on the `Host` header.

1. Go to Load Balancer → Routing Policies → Create Routing Policy.
2. Give the policy a name (e.g., `route-api-by-host`).
3. Add a Rule (Rule 1):
   - Match type: `If all match`
   - Condition type: `Header`
   - Operator: `Equals`
   - Key: `host`
   - Value: the subdomain (e.g., `api.example.com`) — do not include protocol
4. Under Actions, choose the backend set to forward to (e.g., `backend-api`).
5. Save the rule and complete the policy creation.

Repeat steps to create one policy per subdomain/backend combination (Rule 1, Rule 2, etc.).

Tips:
- You can add multiple conditions in a rule (e.g., path and host).
- For complex routing, combine path and header matches or use multiple rules in a routing policy.

---

## 5) Attach Routing Policy to Listener
1. In Load Balancer → Listeners → pick your HTTPS listener → Actions (three dots) → Edit.
2. Under Routing Policy, add the routing policy (or multiple policies) you created.
3. Save changes.

Important:
- The listener (HTTPS) will inspect the Host header and apply the routing policy. Ensure that the listener uses the appropriate SSL certificate if TLS termination is at the load balancer.
- If you require end-to-end encryption, use a TCP listener and configure SSL on backends, or enable TLS passthrough if supported.

---

## 6) Test the subdomain
- In a terminal:
  - Check DNS resolution:
    - dig +short api.example.com
  - Browse the subdomain (https://api.example.com) — it should load content from the backend without NGINX in front.
- If a browser cache or local DNS cache interferes, flush DNS or use curl with the Load Balancer IP and Host header for testing:
  ```bash
  curl -k -H "Host: api.example.com" https://<load-balancer-ip>/
  ```
  or for HTTP listener:
  ```bash
  curl -H "Host: api.example.com" http://<load-balancer-ip>/
  ```

---

## Troubleshooting
- Subdomain resolves but gets wrong backend:
  - Confirm routing policy value exactly matches the Host header (case-insensitive match is common, but avoid extra spaces).
  - Verify the policy is attached to the correct listener.
- Backends show unhealthy:
  - Check that the backend port set on the backend entry matches the service port.
  - Confirm the health check configuration matches the backend service behavior.
  - Use `telnet <backend-ip> <port>` or `nc -zv` from a test host in the same network to validate reachability.
- DNS changes not applied:
  - Confirm A record points to the Load Balancer IP and TTL has expired; use `dig` to verify propagation.
- SSL/TLS issues:
  - Ensure the listener certificate matches the domain or use a SAN/wildcard certificate that covers subdomains.
  - If using TLS passthrough or TCP listeners, configure backend certs appropriately.

---

## Notes & Best Practices
- Use separate backend sets for services with different ports or scaling requirements.
- Prefer path-based routing in combination with host-based routing for multi-tenant or multi-service architectures.
- Keep DNS TTL low during testing so changes propagate quickly, then raise TTL for production.
- Use Network Security Groups (NSGs) or Security Lists to restrict which source IPs can reach the Load Balancer or backends.
- Monitor Load Balancer logs and backend health checks regularly.
- If you previously had NGINX performing routing or TLS termination, migrate those responsibilities to the Load Balancer carefully: export certs, update DNS, and remove NGINX after confirming traffic flows correctly.

---

## Example flow (quick)
1. Create `backend-api` backend set with TCP health check port 8080.
2. Add backend instances with backend port 8080.
3. Create hostname: `api.example.com`.
4. Create routing policy `route-api` matching header `host` equals `api.example.com` → action: forward to `backend-api`.
5. Attach `route-api` to HTTPS listener.
6. Create DNS A record for `api.example.com` pointing to load balancer IP.
7. Test with `curl -H "Host: api.example.com" https://<lb-ip>/`.
