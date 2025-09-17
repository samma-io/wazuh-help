
# Securing Wazuh: A Guide to Keycloak OAuth2 & MFA Integration

If you're running Wazuh in an environment with compliance requirements like **PCI DSS**, **SOC 2**, or **HIPAA**, you know that **multi-factor authentication (MFA)** and strict access control are non-negotiable. The need to prove who has access to sensitive security logs—and who can modify configurations—means that shared accounts and simple passwords are no longer enough.

It's time to connect Wazuh to a modern authentication provider. This guide will walk you through integrating Wazuh with **Keycloak**, a powerful open-source Identity and Access Management (IAM) solution.

Connecting Keycloak to Wazuh accomplishes two key goals:

1.  **Elevates Security & Compliance**: It immediately brings Wazuh's authentication up to modern standards with support for SSO, MFA, and centralized user management.
2.  **Improves Usability**: It allows you to grant role-based access to more people on your team, like developers and analysts, without managing separate credentials for each tool. By centralizing identity, everyone can log in and view the alerts relevant to them securely.

-----

## Step 1: Deploying Keycloak with Docker Compose

There are many ways to set up Keycloak, including Helm charts for Kubernetes. For this guide, we'll use a simple `docker-compose` file to get a server running quickly. This setup includes a PostgreSQL database for persistence.

Create a file named `docker-compose.yml`:

```yaml
version: '3.8'

services:
  postgres:
    image: postgres:16.2
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    networks:
      - keycloak_network

  keycloak:
    image: quay.io/keycloak/keycloak:23.0.6
    command: start
    environment:
      KC_HOSTNAME: localhost
      KC_HOSTNAME_PORT: 8080
      KC_HOSTNAME_STRICT_BACKCHANNEL: 'false'
      KC_HTTP_ENABLED: 'true'
      KC_HOSTNAME_STRICT_HTTPS: 'false'
      KC_HEALTH_ENABLED: 'true'
      KEYCLOAK_ADMIN: ${KEYCLOAK_ADMIN}
      KEYCLOAK_ADMIN_PASSWORD: ${KEYCLOAK_ADMIN_PASSWORD}
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://postgres/${POSTGRES_DB}
      KC_DB_USERNAME: ${POSTGRES_USER}
      KC_DB_PASSWORD: ${POSTGRES_PASSWORD}
    ports:
      - "8080:8080"
    restart: always
    depends_on:
      - postgres
    networks:
      - keycloak_network

volumes:
  postgres_data:
    driver: local

networks:
  keycloak_network:
    driver: bridge
```

This `docker-compose.yml` file references environment variables for your secrets. Create a `.env` file in the same directory to store them:

```
# .env file
POSTGRES_DB=keycloak
POSTGRES_USER=keycloak
POSTGRES_PASSWORD=your_strong_postgres_password

KEYCLOAK_ADMIN=admin
KEYCLOAK_ADMIN_PASSWORD=your_strong_admin_password
```

Now, launch Keycloak by running:

```bash
docker-compose up -d
```

You should be able to access the Keycloak admin console at `http://localhost:8080`.

-----

## Step 2: Configuring the Keycloak Realm for Wazuh

Setting up Keycloak is the easy part. Now we need to configure it to act as an authentication provider for Wazuh. This involves creating a client, roles, and groups.

1.  **Create a Client**: In the Keycloak admin console, create a new client for Wazuh. Let's call the **Client ID** `wazuh`. Ensure the client has **Standard flow** enabled and add your Wazuh dashboard URL to the **Valid Redirect URIs** (e.g., `https://your-wazuh-host.com/*`).

2.  **Create Realm Roles**: We need roles that will correspond to permissions in Wazuh. Go to **Realm Roles** and create two roles:

      * `wazuh_admin`
      * `wazuh_user`

3.  **Create Groups**: Using groups is the best way to manage user permissions. Go to **Groups** and create two groups that mirror the roles:

      * `wazuh_admins`
      * `wazuh_users`

4.  **Map Roles to Groups**: This is the crucial step.

      * Click on the `wazuh_admins` group.
      * Go to the **Role Mappings** tab.
      * Assign the `wazuh_admin` realm role to this group.
      * Repeat the process for the `wazuh_users` group, assigning it the `wazuh_user` role.

Now, when you add a user to the `wazuh_admins` group, they will automatically inherit the `wazuh_admin` role, which Wazuh will use to grant them permissions.

> **Shortcut**: To make this easier, I have exported my Keycloak realm setup. You can download the `realm-export.json` file from [this GitHub repository](https://www.google.com/search?q=https://github.com/example/repo) and import it into your Keycloak instance.

-----

## Step 3: Configuring Wazuh to Use Keycloak

It's time to tell Wazuh to use Keycloak for authentication. This is done by editing the OpenSearch Dashboards configuration file.

SSH into your Wazuh server and edit `/etc/wazuh-dashboard/opensearch_dashboards.yml`. Find the `opensearch_security.auth.type: "basicauth"` line and replace it with the OpenID configuration below.

> **Important**: Before editing, make a backup of this file\!
> `cp /etc/wazuh-dashboard/opensearch_dashboards.yml /etc/wazuh-dashboard/opensearch_dashboards.yml.bak`

```yaml
# /etc/wazuh-dashboard/opensearch_dashboards.yml
opensearch_security.auth.type: "openid"
opensearch_security.openid:
  connect_url: "http://YOUR_KEYCLOAK_IP:8080/realms/master/.well-known/openid-configuration"
  client_id: "wazuh"
  client_secret: "YOUR_CLIENT_SECRET_FROM_KEYCLOAK" # Find this in your 'wazuh' client credentials tab
  scope: "openid profile email"
  header: "Authorization"
  subject_key: "preferred_username" # Tells Wazuh to use the Keycloak username
  roles_key: "roles" # Tells Wazuh to look for the 'roles' claim in the token
```

  * Update the `connect_url` to point to your Keycloak server's IP or hostname.
  * Retrieve your `client_secret` from the **Credentials** tab of your `wazuh` client in Keycloak.

After saving the file, restart the Wazuh dashboard to apply the changes:

```bash
systemctl restart wazuh-dashboard
```

-----

## Step 4: Mapping Roles in the Wazuh Dashboard

The final step is to tell Wazuh what permissions the `wazuh_admin` and `wazuh_user` roles from Keycloak should have.

1.  Log into your Wazuh dashboard using your old local admin account.
2.  Navigate to the menu icon **☰ \> Security \> Roles**.
3.  Click on the `all_access` role.
4.  Go to the **Mapped users** tab.
5.  Under **Backend roles**, add `wazuh_admin`. This maps anyone with the Keycloak `wazuh_admin` role to Wazuh's super-admin role.
6.  Click **Map**. You can create more granular roles for your `wazuh_user` group as needed.

Now, when a user from your `wazuh_admins` group in Keycloak logs into Wazuh, they will be granted full administrative access.

-----

## Troubleshooting Tips

The most common issue is a misconfiguration in the JWT token. If the roles aren't being passed correctly, the mapping will fail.

You can verify the token directly in Keycloak:

1.  Go to **Clients \> `wazuh` \> Client Scopes**.
2.  Click on the **Evaluate** tab.
3.  Select a user you've placed in a `wazuh_` group.
4.  Click **Evaluate** and inspect the **Generated Access Token**. In the decoded token, you should see a `roles` array containing either `wazuh_admin` or `wazuh_user`. If it's not there, your group-to-role mapping is incorrect.

-----

## A Quick Note on Production Security 🔒

The Docker Compose and Wazuh configurations in this guide use **HTTP** for simplicity. **In a production environment, you must configure proper TLS/HTTPS for both Keycloak and Wazuh.** Exposing your authentication system over an unencrypted channel is a major security risk.
