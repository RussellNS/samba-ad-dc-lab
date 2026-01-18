# Samba AD DC Lab Provisioner

This repository provides a containerized **Samba Active Directory Domain Controller (AD DC)** specifically designed for **lab rollouts, automated testing, and rapid prototyping**.

## What This Does

This setup automates the deployment of a Samba-based Domain Controller. It handles the initial provisioning, raises the functional level to **2016**, updates the AD schema to **2019**, configures reverse DNS lookup zones automatically, and allows for the bulk creation of user accounts during the first boot.

NOTE: The AD “functional level” is the same across Windows Server 2016, 2019, 2022, and 2025.

---

## Use Case: Lab vs. Production

While there are generalized Samba containers available—most notably the excellent [docker-samba-dc by burnbabyburn](https://github.com/burnbabyburn/docker-samba-dc)—this repository is specialized for **speed and automation in non-production environments.**

Use this repo if you need a "domain-in-a-box" that is fully populated with users and ready for testing in minutes. For long-term production stability without the automated lab-provisioning logic, the general-purpose repo mentioned above is recommended.

## Configuration & Environment Variables

The setup uses a specific hierarchy for configuration variables. You can provide these through standard Environment Variables, a mounted environment file, or Docker Secrets.

### Variable Definitions

| Variable        | Required? | Description                                                                                                                                                                                  | Example                                                  |
|:----------------|:----------|:---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|:---------------------------------------------------------|
| `AD_FQDN`         | **Yes**       | The Full Quantified Domain Name.<br />Other required variables, like REALM or NETBIOS are derived from this. For example, ‘example.com’ becomes Realm: ‘EXAMPLE.COM’ and NetBIOS: ‘EXAMPLE’. | `vlab.test`                                                |
| `AD_ADMIN_PASS`   | **Yes**       | The password for the default Administrator account.                                                                                                                                          | `Super-Seekrit-Pass1`                                      |
| `DNS_FORWARDER`   | **Yes**       | Upstream DNS server for non-AD queries.                                                                                                                                                      | `8.8.8.8` or ‘`none'`                                        |
| `AD_CREATE_USERS` | No        | Comma-separated list for auto-provisioning.                                                                                                                                                  | `testUser1:pass:Domain Admins, testUser2:pass:Domain User` |
| `TZ`              | No        | Set the container timezone.                                                                                                                                                                  | `US/Central`                                               |

### Variable Declaration

This container supports three ways to provide configuration information. They are processed in a specific order of precedence:

**Logic:** `Standard ENV` -> `provision.env` -> `Docker Secrets`

Each stage overwrites the previous one. This allows you to set defaults in the Compose file but override them with secure secrets.

#### 1. Standard ENV Variables

Defined directly in your `compose.yaml` or `docker run` command.

* **Pros:** Easy to set up.
* **Cons:** Values are visible in `docker inspect`, `docker logs`, and build history.

#### 2. Provisioning File (`provision.env`)

You can temporarily mount a file containing your variables to `/tmp/provision.env`.

* **Pros:** Keeps credentials out of the Docker process environment.
* **Example File Content:**

  ```env
  AD_ADMIN_PASS=MySecurePass123
  AD_FQDN=example.com
  DNS_FORWARDER=1.1.1.1
  AD_CREATE_USERS=tester:Pass123:Domain Admins
  
  ```

#### 3. Docker Secrets

The most secure method. Values are stored in temporary files within `/run/secrets/` by the Docker engine.

* **Pros:** Sensitive data is never stored in the container's environment variables or logs.
* **Setup:** Create a text file for each variable (e.g., `ad_admin_pass.txt` containing only the password) and define them in your `compose.yaml`. The entrypoint script will automatically look for secrets matching the variable names.

---

## Deployment Examples

### Docker Run Example

Adjust the following CLI command to match your environment:

```bash
docker run -d \
  --name samba-ad-dc \
  --cap-add SYS_ADMIN \
  --network macvlan_net \
  --ip 10.10.10.11 \
  -e AD_ADMIN_PASS="P@55w0rd1!" \
  -e AD_FQDN="example.com" \
  -e DNS_FORWARDER="8.8.8.8" \
  -v samba_config:/etc/samba \
  -v samba_data:/var/lib/samba \
  -f ./Dockerfiles/Dockerfile_samba-ad-dc
```

### Docker Compose (`compose.yaml`)

If you prefer using Compose over the CLI, adjust the following YAML to match your environment:

```yaml
services:
  dc1:
    build:
      context: .
      dockerfile: ./Dockerfiles/Dockerfile_samba-ad-dc
    container_name: samba-ad-dc
    hostname: dc1
    cap_add:
      - SYS_ADMIN
    networks:
      macvlan_net:
        ipv4_address: 192.168.1.123
    environment:
      - TZ=US/Central
      - AD_ADMIN_PASS=P@55w0rd1!
      - AD_FQDN=example.com
      - DNS_FORWARDER=8.8.8.8
      - AD_CREATE_USERS=testUser1:Pass123:Domain Admins,testUser2:Pass456:Domain Users
    volumes:
      - config_vol:/etc/samba
      - data_vol:/var/lib/samba

volumes:
  config_vol:
  data_vol:

networks:
  macvlan_net:
    external: true
```

---

## Networking Requirements

### Macvlan & Static IP

While this container can run on a standard Docker bridge, it is **strongly recommended** to use a Docker `macvlan` network with a dedicated IP. Active Directory relies heavily on DNS and Kerberos, which often require the container to have its own unique MAC address and IP on your physical/virtual network to avoid port conflicts and discovery issues.

---

## Volume Mounts

To ensure your Active Directory database persists across container restarts and updates, you must mount the following volumes:

* `/etc/samba`: Stores the generated `smb.conf` and core configuration files.
* `/var/lib/samba`: Stores the actual AD database (LDB files), sysvol, and private keys.
* `/tmp/provision.env` **(Optional/Temporary)**: This is a "bind mount" used only during the first run. Once the provisioning process is complete and the `smb.conf` exists, you can stop the container and remove this mount from your configuration if desired.

---

## Sample Use Cases

The ability to stand up and tear down a clean AD/DC with pre-provisioned users has multiple use cases:

* Application testing that requires authenticating or joining to an AD domain
* Automation testing that reads or modifies AD or LDAP attributes
* Classroom lab environments
* Home lab environments

### Managing This Container

The easiest way to manage the AD/DC deployed from this container, is to join a Windows client to the domain, and use the Remote Server Administration Tools (RSAT).

#### Joining to the Domain

To join a Windows client (10, 11, Server 20xx), follow these steps.

1. Deploy a Windows client in your lab environment.
2. Update the DNS settings of the Windows client to the IP of the AD/DC deployed by this container.
3. Join the Windows client to the domain.

#### Using RSAT

1. Install RSAT.
   * **Windows 10/11**

   ```PowerShell
   Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\Servicing" -Name "RepairContentServerSource" -Value 2 -Force -ErrorAction SilentlyContinue
   ```
   * **Windows Server 19xx/20xx**

   ```PowerShell
   Get-WindowsFeature | Where-Object { $*.Name -like "RSAT*" -and -not $*.Installed } | Install-WindowsFeature -IncludeManagementTools
   ```
2. Use RSAT.

NOTE: All the RSAT tools will work except DHCP.  The DHCP service that AD/DC’s run is tightly paired to AD services, so it may “feel” like AD/DC’s and DHCP are coupled, especially with backend API’s to allow for dynamic DNS updates.  However, DHCP is a completely separate service.  Standing up DHCP is outside the scope of this container.

---

## Verification & Health

Once the container is running, you can monitor the status and verify the Active Directory configuration using the following methods:

### Check Functional Level

This setup automatically raises the functional level to **2016**. To verify this:

```bash
docker exec samba-ad-dc samba-tool domain level show
```

### Health Monitoring

The container includes an integrated health check that goes beyond simple process monitoring. It performs a functional LDAP query against the local instance:
`samba-tool domain info 127.0.0.1`

If the directory service stops responding or the database becomes corrupted, the container status will be marked as `unhealthy`.

### Cleanup & Security

Immediately following the provisioning of the domain, the entrypoint script executes a cleanup function. This wipes sensitive provisioning variables (like `AD_ADMIN_PASS` and user passwords) from the shell memory of the running PID 1 process to minimize the exposure of credentials in the running environment.

---

## Troubleshooting

* **DNS Resolution:** Ensure your client machines (Windows/Linux) are set to use the container's static IP as their primary DNS server. AD will not function correctly if clients cannot resolve the SRV records.
* **Macvlan Issues:** If the container cannot reach the gateway or other hosts on the same subnet, ensure your Docker host allows sub-interfaces on the physical NIC and check for IP/MAC conflicts on the network.
* **Logs:** For detailed startup information and to verify the provisioning steps:

  ```bash
  docker logs -f samba-ad-dc
  ```

## Disclaimer

This setup is intended for **lab and testing purposes only**. It requires elevated privileges (`SYS_ADMIN`) and uses automated credential handling that may not meet the security requirements of a hardened production environment. Use at your own risk.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

```
```
