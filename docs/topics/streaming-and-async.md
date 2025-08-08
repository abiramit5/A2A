# Streaming and Asynchronous Operations for Long-Running Tasks

The Agent2Agent (A2A) protocol supports long-running tasks that may involve
multiple steps, incremental results, or human intervention. A2A provides
mechanisms for managing these asynchronous interactions, allowing clients to
receive updates regardless of their connection state.

## Streaming with Server-Sent Events (SSE)

A2A uses Server-Sent Events (SSE) for real-time communication of incremental
results, ongoing status updates, or when clients maintain active HTTP connections
to the A2A Server.

### Key Characteristics

-   **Initiation:** Clients use `message/stream` to send a message and subscribe
    to updates.
-   **Server capability:** Servers indicate streaming support with
    `capabilities.streaming: true`.
-   **Connection:** Successful subscription results in an HTTP 200 OK response
    with `Content-Type: text/event-stream`, keeping the connection open for
    events.
-   **Event structure:** The server sends events containing a JSON-RPC 2.0
    Response object (`SendStreamingMessageResponse`), with the `id` matching the
    client's request.
-   **Event types (within `SendStreamingMessageResponse.result`):** The event
    types include:
    - [`Task`](../specification.md#61-task-object): Current task state.
    - [`TaskStatusUpdateEvent`](../specification.md#722-taskstatusupdateevent-object):
        Changes in task state, including intermediate messages.
    - [`TaskArtifactUpdateEvent`](../specification.md#723-taskartifactupdateevent-object):
        New or updated artifacts, with `append` and `lastChunk` for
        reassembly.
-   **Stream termination:** The server sets `final: true` in a
    `TaskStatusUpdateEvent` to signal the end of updates for a request, and
    closes the SSE connection.
-   **Resubscription:** Clients can use `tasks/resubscribe` to reconnect if
    the connection breaks before a `final: true` event.

**When to Use Streaming:**

- For real-time progress monitoring.
- For incremental delivery of large results.
- For interactive conversations with immediate feedback.
- For low-latency updates.

For more information, refer to the Protocol Specification for detailed structures:

- [`message/stream`](../specification.md#72-messagestream)
- [`tasks/resubscribe`](../specification.md#79-tasksresubscribe)

## Push Notifications for Disconnected Scenarios

A2A supports push notifications for long-running tasks or when clients prefer
not to maintain persistent connections. The A2A Server notifies a client's
webhook when a significant task update occurs.

### Key Characteristics

- **Server capability:** Servers indicate support with
    `capabilities.pushNotifications: true`.
- **Configuration:** Clients provide a
    [`PushNotificationConfig`](../specification.md#68-pushnotificationconfig-object).
    - Can be included in `message/send` or `message/stream` requests, or
        set separately with `tasks/pushNotificationConfig/set`.
    - Includes:
        - `url`: The webhook URL for notifications.
        - `token`: An optional client-generated token for validation.
        - `authentication`: Optional authentication details for the server to use.
- **Notification Trigger:** Sent on significant state changes (e.g.,
    terminal, `input-required`, or `auth-required`).
- **Notification payload:** The notification should contain the `Task ID` and
    the new `TaskState`.
- **Client action:** Clients use `tasks/get` with the `task ID` from the
    notification to retrieve the updated `Task` object.

### The Push Notification Service (Client-Side Webhook Infrastructure)

- The `PushNotificationConfig.url` points to a client-side service that
    receives HTTP POST notifications.
- Its responsibilities include:
    - Authenticating the notification from the A2A server.
    - Validating the notification's relevance to the client.
    - Relaying the notification to the appropriate client system.
- Can be a simple endpoint in local development or a robust service in
    production.

### Security Considerations for Push Notifications

Security is critical for push notifications due to their asynchronous and
outbound nature. Both the A2A Server and the client's webhook receiver have
responsibilities.

#### A2A Server Security (When Sending Notifications to Client Webhook)

1.  **Webhook URL validation:**
    - Servers **SHOULD NOT** trust client-provided URLs.
    - **Mitigation strategies:**
        - Allowlisting trusted domains.
        - Ownership verification with a `validationToken`.
        - Using network controls to restrict outbound requests.
2.  **Authenticating to the client's Webhook:**
    - The A2A Server **MUST** authenticate according to
        `PushNotificationConfig.authentication`.
    - Common authentication schemes for server-to-server webhooks include:
        - Bearer Tokens (OAuth 2.0).
        - API Keys.
        - HMAC Signatures.
        - Mutual TLS (mTLS).

#### Client Webhook Receiver Security (When Receiving Notifications from A2A Server)

1.  **Authenticating the A2A server:**
    - The webhook **MUST** verify the authenticity of incoming requests.
    - **Verify signatures/tokens:**
        - Validate JWT signatures against public keys.
        - Verify HMAC signatures.
        - Validate API keys.
    - **Validate `PushNotificationConfig.token`:** Check for the provided
        token in a custom header.
2.  **Preventing replay attacks:**
    - **Nonces/Unique IDs:** For critical notifications, consider using unique,
        single-use identifiers (nonces or event IDs) for each notification. The
        webhook should track received IDs (for a reasonable window) to prevent
        processing duplicate notifications. A JWT's `jti` (JWT ID) claim can
        serve this purpose.
3.  **Secure key management and rotation:**
    - If using cryptographic keys (symmetric secrets for HMAC, or asymmetric
        key pairs for JWT signing/mTLS), implement secure key management
        practices, including regular key rotation.
    - For asymmetric keys where the A2A Server signs and the client webhook
        verifies, protocols like JWKS (JSON Web Key Set) allow the server to
        publish its public keys (including new ones during rotation) at a
        well-known endpoint. Client webhooks can then dynamically fetch the
        correct public key for signature verification, facilitating smoother key
        rotation.

##### Example Asymmetric Key Flow (JWT + JWKS)
1.  The client sets `PushNotificationConfig` specifying `authentication.schemes:
    ["Bearer"]` and possibly an expected `issuer` or `audience` for the JWT.
2.  A2A Server, when sending a notification:
    - Generates a JWT, signing it with its private key. The JWT includes
        claims like `iss` (issuer), `aud` (audience - the webhook), `iat`
        (issued at), `exp` (expires), `jti` (JWT ID), and `taskId`.
    - The JWT header (`alg` and `kid`) indicates the signing algorithm and key
        ID.
    - The A2A Server makes its public keys available via a JWKS endpoint (URL for this endpoint might be known to the webhook provider or discovered).
3.  The client webhook, upon receiving the notification:
    - Extracts the JWT from the `Authorization` header.
    - Inspects the `kid` in the JWT header.
    - Fetches the corresponding public key from the A2A Server's JWKS
        endpoint (caching keys is recommended).
    - Verifies the JWT signature using the public key.
    - Validates claims (`iss`, `aud`, `iat`, `exp`, `jti`).
    - Checks the `PushNotificationConfig.token` if provided.

This comprehensive, layered approach to security for push notifications helps
ensure that messages are authentic, integral, and timely. It also protects both the
sending A2A Server and the receiving client webhook infrastructure.

