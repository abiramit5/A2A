# Agent Discovery in A2A

To collaborate using the Agent2Agent (A2A) protocol, AI agents must first find each other and understand their capabilities. A2A standardizes agent self-descriptions through the **[Agent Card](../specification.md#5-agent-discovery-the-agent-card)**. However, discovery methods for these Agent Cards vary by environment and requirements. The Agent Card defines what an agent offers. Various strategies exist for a client agent to discover these cards. The choice of strategy depends on the deployment environment and security requirements.

## The Role of the Agent Card

The Agent Card is a JSON document that serves as a digital "business card" for an A2A Server (the remote agent). It is crucial for agent discovery and interaction. The key information included in an Agent Card includes:

- **Identity:** `name`, `description`, `provider` information.
- **Service endpoint:** The `url` for the A2A service.
- **A2A capabilities:** Supported features like `streaming` or `pushNotifications`.
- **Authentication:** Required `schemes` (e.g., "Bearer", "OAuth2").
- **Skills:** Agent's tasks (`AgentSkill` objects) with `id`, `name`, `description`, `inputModes`, `outputModes`, and `examples`.

Client agents use the Agent Card to determine an agent's suitability, structure requests, and ensure secure communication.

## Discovery Strategies

The following sections detail common strategies employed by client agents to discover remote Agent Cards:

### 1. Well-Known URI

This strategy is ideal for public agents or those requiring broad discoverability within a specific domain.

- **Mechanism:** A2A Servers host Agent Cards at a standardized, 'well-known' URI.
- **Standard Path:** `https://{agent-server-domain}/.well-known/agent-card.json` ([RFC 8615](https://www.ietf.org/rfc/rfc8615.txt)).
- **Process:**
    1. Client agents identify a potential A2A Server's domain (e.g., `smart-thermostat.example.com`).
    2. The client sends an HTTP `GET` to `https://smart-thermostat.example.com/.well-known/agent-card.json`.
    3. The server returns the Agent Card in JSON format, if accessible.
-   **Advantages:** Simple, standardized, and enables automated discovery.
-   **Considerations:** This approach is well-suited for open or domain-controlled discovery. Authentication is sometimes required at the endpoint for Agent Cards containing sensitive information.

### 2. Curated Registries (Catalog-Based Discovery)

In enterprise environments or marketplaces, Agent Cards are typically published to a central registry.

- **Mechanism:** A registry maintains a collection of Agent Cards. Clients can query the registry based on specific criteria (e.g., skills, tags).
- **Process:**
    1. A2A Servers register their Agent Cards with the registry.
    2. Client agents query the registry's API (e.g., to find agents with an "image-generation" skill).
    3. The registry returns the matching Agent Cards or references.
- **Advantages:**
    - Centralized management and governance.
    - Discovery based on capabilities, not just domain names.
    - Access controls and trust mechanisms.
    - Company-specific or public marketplaces.
- **Considerations:** Requires a registry service. The A2A specification does not define a standard API for registries.

### 3. Direct Configuration / Private Discovery

For tightly coupled systems, private agents, or development purposes, clients can be directly configured with Agent Card information or URLs.

-   **Mechanism:** Client applications can utilize hardcoded details, configuration files, environment variables, or proprietary APIs for discovery.
-   **Process:** Application-specific.
-   **Advantages:** This method is straightforward for known, static relationships.
-   **Considerations:** This approach is inflexible for dynamic discovery scenarios. Changes to Agent Card information necessitate client reconfiguration. Furthermore, proprietary API-based discovery lacks standardization.

## Securing Agent Cards

Agent Cards can contain protected information, including:

-   The `url` of an internal or restricted agent.
-   `authentication.credentials` details (e.g., OAuth token URL). **Secrets must not be stored within Agent Cards.**
-   Descriptions of sensitive skills.

**Protection Mechanisms:**

- **Endpoint access control:** Secure the HTTP endpoint serving the Agent Card (e.g., `/.well-known/agent-card.json`, registry API) if not for public access.
    -   **mTLS:** Require mutual TLS (mTLS).
    -   **Network restrictions:** Restrict access to specific IP ranges or networks.
    -   **Authentication:** Require HTTP authentication (e.g., OAuth 2.0).
- **Selective disclosure by registries:** Registries can implement selective disclosure, returning different Agent Cards based on client identity and permissions.

If an Agent Card contains sensitive data (which is **not recommended** for secrets), it **must** be protected with robust authentication and authorization mechanisms. The A2A specification encourages the use of out-of-band dynamic credentials rather than embedding static secrets within the Agent Card.

## Future Considerations

The A2A community explores standardizing registry interactions or advanced discovery protocols. Feedback is welcome to enhance A2A agent discoverability and interoperability.
