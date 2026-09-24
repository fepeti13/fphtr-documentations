# Recognition communication: first approach

Status: **initial approach** (decided 2026-09-24). It can change as we learn more.

Connection details for the university server (VPN, SSH, credentials) are in `university-server-connection.md`.

## Decision

- **No FastAPI or any other running service on the university server.**
- A model is loaded inside an **SSH session** on the university server. All further recognition requests are sent through **the same SSH connection**.
- If the SSH connection drops, it is fine that the model is dropped too. This is intended: it prevents a stale model staying loaded on the university server.

## Flow

```
Caller (for example the UI)
   |  HTTP endpoints
   v
Local service (on the user's machine)
   |  converts each call to commands over one SSH connection
   v
University server (MLHub): SSH session with the model loaded
```

- The **local service** runs on the user's machine and exposes endpoints.
- It takes the incoming calls and converts them to SSH calls.
- Example endpoints (illustrative, not final): load the model, recognize the given words/images.

## Consequences

- Nothing has to be deployed or kept alive on the university server.
- The model's lifetime equals the SSH session's lifetime.
- After a dropped connection, the model has to be loaded again.

## Open questions (TBD)

- How requests and responses are framed over the SSH connection.
- How images (crops) are transferred to the server.
- How the local service gets the session username and password, since the password changes with every new MLHub session.
- What the local service does when the connection drops (report an error, or reconnect and reload the model).
- How base and large models are selected.
- Whether requests are handled one at a time or in parallel.
- Session and idle timeouts of MLHub (not documented).
- Which endpoints the caller uses and where cropping happens. This belongs to the UI and recognition interface contract.

## Related Jira stories

- TM-24: connection possibilities for the university server
- TM-26: interface contract between the UI server and recognition server
- TM-29: initial version of the recognition server
