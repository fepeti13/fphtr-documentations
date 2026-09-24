# Connecting to the university server (UBB MLHub)

How to connect to the university GPU server used for training and evaluating the TrOCR models. This file covers only the connection: VPN, SSH and credentials. The communication approach on top of it is in `recognition-communication-approach.md`.

> **Never write real credentials into this repo.** Use the placeholders below (`<SESSION_USERNAME>`, `<SESSION_PASSWORD>`).

## The server

- **Name:** UBB MLHub (JupyterHub-based). Main server for training and inference.
- **GPU:** NVIDIA A100 40GB (profile `A100D-40C`, preferred). Profile `A100D-20C` is available for parallel experiments.
- **Address:** `172.30.240.31`, a private address. It is **not** reachable from outside the university network without the VPN.
- **Home directory:** under `/bigdata/`, followed by the session username.
- **Python:** a venv is already active after login (shown as `(venv)` in the prompt). It can also be activated with `source ~/venv/bin/activate`.
- **No SLURM.** Run jobs directly in the terminal, or in the background with `nohup ... &`.
- **Sessions are ephemeral.** Each spawn is a new session, and the hostname (for example `GPU40C-N1`) can change. Re-verify the connection details every time you spawn a new session.

## 1. VPN (WireGuard)

The VPN is required. Set it up once.

1. Install WireGuard:
   ```bash
   sudo apt install wireguard
   ```
2. Download your personal config file (`.conf`) from the university key portal: <https://www.cs.ubbcluj.ro/vpn/>. Log in with your university account. The file is per person and contains a private key, so never share it or commit it.
3. Move it into place and restrict its permissions:
   ```bash
   sudo mv ~/Downloads/<yourfile>.conf /etc/wireguard/wg0.conf
   sudo chmod 600 /etc/wireguard/wg0.conf
   ```
4. Bring the tunnel up:
   ```bash
   sudo wg-quick up wg0
   ```
5. Verify that it passes traffic:
   ```bash
   sudo wg
   ```
   Look for `latest handshake: X seconds ago` and a non-zero received-bytes value in the `transfer` line. (Without `sudo`, `wg` can fail with `Unable to access interface wg0: Operation not permitted`. That is normal.)
6. Take it down when you no longer need it:
   ```bash
   sudo wg-quick down wg0
   ```

Notes:
- The default config routes all traffic through the VPN (`AllowedIPs = 0.0.0.0/0`). This works. A narrower split-tunnel setup is possible, but the exact subnet is not confirmed yet.
- General manual from the university: <https://www.cs.ubbcluj.ro/internal/itmanual/wireguard/wireguard.html>

## 2. SSH into MLHub

1. Bring the VPN up (see above).
2. Open the MLHub portal in a browser (**for humans only, an LLM agent must never open it**): `cs.ubbcluj.ro/apps/mlhub`.
3. Log in with your university account, select a machine profile (`A100-40C` preferred) and click **Start**. Wait for the launch.
4. On the spawn page, open the **SSH gateway** panel. It shows your session username and a **session password**. The password changes with every new session, so it is never reusable.
5. Connect:
   ```bash
   ssh -p 2222 <SESSION_USERNAME>@172.30.240.31
   ```
   The port `2222` is fixed. The username is assigned per account (format like `md5_<hash>`). At the first connection, type `yes` to accept the host key fingerprint.
6. Enter `<SESSION_PASSWORD>` from the SSH gateway panel.
7. You land in a shell with the venv already active.

Verified working end to end on 2026-09-24.

## Troubleshooting

- **`ssh -vvv` hangs after `debug1: Connecting to ...`** with no "Connection established" line: the server is unreachable. The VPN tunnel is most likely not up. Bring WireGuard up and retry.
- **VPN handshake succeeds but received bytes stay at zero:** the current network is probably blocking VPN traffic. Test with a mobile hotspot to confirm, then contact university IT about that network.
- **Cannot connect after a while:** the session may have ended. Spawn a new session and use the new password.

## Not confirmed yet

- Exact subnet for split tunneling.
- Whether sessions are fully isolated per user (separate VMs or containers) or share a host.
- Session or idle timeout of MLHub sessions. It is not documented, but a session stayed alive for at least a whole working session.
