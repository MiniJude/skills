---
name: "cloud-codex-installer"
description: "Interactively installs and verifies Codex on a cloud server. Invoke when the user wants you to ask step by step, connect remotely, install dependencies, and keep going until Codex works."
---

# Cloud Codex Installer

Use this skill when the user wants you to actively drive the whole cloud-server setup conversation: ask for connection info step by step, attempt login, detect the environment yourself, install Codex yourself, and keep the user informed at every stage.

## Core Behavior

This skill must feel like an operator doing the work live, not like a form for the user to fill out.

The agent must:

1. Ask for only the next piece of information needed right now.
2. Perform the next action immediately after getting that information.
3. Detect the server environment by itself after login.
4. Decide what dependencies are missing by itself.
5. Explain each step before or while doing it.
6. Continue until Codex is actually usable or a real blocker is identified.

The agent must not:

- Ask the user to provide a full template of server details up front.
- Ask the user to manually inspect the OS, Node.js, npm, PATH, or dependency state.
- Ask the user whether common dependencies are installed if the agent can check directly after login.
- Ask the user to send the SSH password in chat.
- Ask for the password before the connection attempt is about to happen.

## Mandatory Interaction Style

The workflow must be conversational and incremental.

### Step 1: Ask For IP First

Start by asking only for the cloud server IP address.

Do not ask for password, OS version, Node.js status, or other environment details in the first message.

### Step 2: Clarify Username Or Port Only If Needed

After receiving the IP:

- Assume username `root` unless the user says otherwise.
- Assume SSH port `22` unless the user says otherwise.

If those assumptions may be wrong, confirm them briefly. Do not ask for unrelated details.

### Step 3: Launch A Terminal For Password Entry

Right before attempting authentication, tell the user that you are about to connect, launch an interactive terminal session, and let the user type the SSH password directly into that terminal.

Do not ask the user to paste the password into chat.
Do not bundle password handling into the initial information-gathering step.

### Step 4: Continue With Live Progress Updates

As the work proceeds, keep the user informed in a transparent way:

- What you are doing now
- What you just discovered
- Why the next step is needed
- Whether a failure happened and how you are responding

## Transparency Rules

The user explicitly wants visible execution progress.

Therefore, before every major action, give a short progress update such as:

- starting connection
- verifying remote access
- detecting OS and package manager
- checking Node.js and npm
- installing missing dependencies
- installing Codex
- verifying `codex --version`
- re-checking in a fresh shell

When a command fails:

- Say what failed in plain language
- Include only non-secret error details
- Explain the next troubleshooting step

Do not hide the workflow behind vague messages like “processing” or “working on it”.

## Secret Handling

- Never store passwords or API keys in files, memory, logs, or summaries.
- Never repeat the user's password back to them.
- For SSH passwords, prefer terminal-only entry by the user instead of chat collection.
- Use secrets only for the immediate connection or authentication step.
- If Codex later needs an API key or login token, ask only when that exact step is reached.

## Connection Strategy

The local environment is Windows PowerShell. Prefer direct SSH in an interactive terminal when password entry is required, so the user can type the password into the terminal instead of sending it through the model.

Important IDE caveat: in some sandboxed IDE environments, a terminal session that successfully logs into the remote host cannot be reused as a general remote command channel by the agent, because follow-up commands may be wrapped by local tooling and then fail on the remote machine. Therefore, "login succeeded" does not imply "subsequent tool-driven remote execution will work through that same terminal".

### Preferred Method

1. Start an interactive terminal running `ssh user@host` or `ssh -p <port> user@host`.
2. Tell the user clearly that the terminal is waiting for the SSH password.
3. Let the user enter the password directly into the terminal prompt.
4. Use that terminal primarily to prove connectivity and, if possible, for direct human-visible interaction.
5. If the IDE cannot reliably drive subsequent commands through that remote shell, immediately switch to one-shot remote execution such as `ssh user@host "command"` instead of assuming the same session is controllable.
6. Keep narrating what is being checked and installed.

### Fallback Order

If direct interactive `ssh` is unavailable or unusable:

1. Check whether another interactive SSH client such as `plink` is available.
2. Still prefer a terminal prompt where the user types the password directly.
3. Before using `Posh-SSH` or `plink`, consider whether the IDE sandbox blocks writes to their default local state paths such as `.poshssh`, `.ssh/known_hosts`, or `PUTTY.RND`.
4. If sandbox restrictions block those clients, prefer plain `ssh` with a temporary workspace-local known-hosts file and one-shot remote commands.
5. Only use a model-mediated credential flow if the user explicitly authorizes it.
6. If no workable method exists, explain the blocker clearly and ask the user for the smallest possible follow-up decision.

### Sandbox-Aware SSH Rules

When operating inside a sandboxed local IDE:

- Avoid assuming OpenSSH can write to the default local `known_hosts` path.
- Prefer `ssh -o UserKnownHostsFile=<workspace-temp-file> -o StrictHostKeyChecking=accept-new ...` when the default `.ssh` directory may be restricted.
- Be cautious with `Posh-SSH`, because it may write under a default `.poshssh` directory outside the workspace and fail even before authentication succeeds.
- Be cautious with `plink`, because it may try to create `PUTTY.RND` under blocked local paths.
- If any local SSH helper fails because of local file restrictions, classify it as a local sandbox/tooling issue, not a remote authentication or server issue.
- Do not keep retrying the same blocked client strategy once the failure mode is clear.

## Remote Discovery Rules

Once logged in, the agent must inspect the environment directly instead of asking the user.

At minimum, detect:

- current user
- shell type
- OS family and version
- CPU architecture
- package manager
- Node.js version
- npm version
- whether `codex` already exists
- relevant profile files affecting `PATH`

If more facts are needed, inspect them on the server instead of asking the user unless credentials or policy decisions are required.

Important rule: check whether `codex` already exists before planning any installation. If `codex`, `node`, and `npm` are already present and healthy, skip installation and move directly to verification.

## Dependency Handling

After discovery, determine which prerequisites are missing.

Typical checks include:

- `node`
- `npm`
- shell startup files
- permissions needed for install

If Node.js or npm is missing, install them using the package manager appropriate to the detected system.

Examples include:

- `apt`
- `dnf`
- `yum`
- `apk`
- `zypper`

If the environment is unusual or package installation fails, consult official Codex requirements before choosing the next step.

## Codex Installation Rules

Treat Codex as the official Codex CLI unless the user explicitly requests something else.

The agent should:

1. Check whether Codex is already installed.
2. Install or update Codex using the current official method.
3. Ensure the resulting binary is reachable from `PATH`.
4. Fix shell profile setup if needed.
5. Re-test in a fresh login shell.

Do not stop after the install command merely exits successfully.
If `codex` is already installed and functional, do not reinstall it just to satisfy the workflow.

## Authentication And Connectivity Rules

After the CLI is installed or discovered, the agent must verify that Codex can actually talk to its backend, not merely start locally.

The agent should:

1. Check whether Codex requires interactive sign-in, API key configuration, or other authentication.
2. Ask for login or API-key help only when the workflow reaches that exact point.
3. Check whether the server can reach the external services required by Codex if requests fail.
4. Distinguish clearly between "CLI present", "CLI starts", "authenticated", and "model can answer".

Important rule: `codex --version` is only an installation check. It is not sufficient to prove real usability.

If authentication is incomplete:

- Do not declare success.
- Tell the user the CLI is installed but not yet authenticated.
- Continue until authentication is completed or a real blocker is found.

If the CLI starts but cannot get a model response:

- Treat that as a failure of real usability.
- Investigate authentication, network reachability, account entitlements, or environment configuration before declaring success.

## Real Smoke Test Rules

A valid smoke test must confirm that Codex returns an actual model response, not just help text, a version string, or a startup banner.

The agent should prefer:

1. A minimal non-destructive prompt.
2. A request with an easy-to-recognize expected response shape.
3. A non-interactive or automation-friendly invocation when available.
4. Repeating that real-response test from a fresh login shell when possible.

Examples of invalid smoke tests:

- `codex --version`
- `codex --help`
- merely launching `codex` and seeing that the binary starts

Examples of valid smoke tests:

- asking Codex to return a short fixed phrase
- asking Codex a tiny factual prompt and confirming a real textual reply is produced

The acceptance criterion is not just exit code `0`. The acceptance criterion is that Codex actually returns meaningful response text.

## Definition Of Done

Only declare success when all of the following are true:

1. Remote `PATH` resolves `codex`.
2. `codex --version` succeeds.
3. Codex authentication is complete, or the exact authentication mode required by the user is confirmed working.
4. A harmless real-response Codex smoke test succeeds and Codex returns actual text.
5. A fresh login shell still finds and runs `codex`.
6. A fresh login shell also succeeds on a real-response smoke test when the transport allows it.

If any of these checks fail, continue troubleshooting.

## Troubleshooting Loop

When verification fails:

1. Describe the current failure to the user.
2. Classify it as environment, dependency, permission, auth, PATH, network, or model-response failure.
3. Apply the smallest safe fix.
4. Tell the user what you changed.
5. Re-run the verification sequence.

Common causes:

- wrong installation method for the OS
- missing PATH export
- unsupported Node.js or npm version
- shell profile not loading
- permission errors
- missing Codex authentication
- remote network restrictions
- Codex can launch but cannot return a real response
- authentication completed locally but not usable in a fresh shell
- local IDE sandbox restrictions on SSH helper state files
- assuming an interactive SSH login terminal is agent-drivable for later commands

## Practical Execution Pattern

In sandboxed IDE environments, this pattern is often safer than trying to "stay inside" one interactive remote shell:

1. Ask for IP.
2. Assume `root` and `22` unless corrected.
3. Launch one interactive `ssh` terminal so the user can type the password directly.
4. Confirm that remote login succeeds.
5. For actual inspection or installation, prefer one-shot remote commands such as `ssh user@host "..."`.
6. Use a workspace-local temp file for host-key storage if default `known_hosts` is blocked.
7. First check whether `codex` is already installed before attempting any install.
8. Only after a real missing dependency is confirmed should the agent install Node.js, npm, or Codex.
9. After installation or discovery, verify authentication state before declaring success.
10. Run a real-response Codex smoke test, not just `--version` or `--help`.
11. Finish with a fresh-shell real-response verification if the transport permits it.

## Communication Pattern

Good behavior looks like this:

1. Ask: “把你的云服务器 IP 发我。”
2. After IP: “我将先按 `root` 和 `22` 尝试；如果不是这组我再调整。”
3. Before login: “下一步我会拉起 SSH 终端，你直接在终端里输入密码，不用在聊天里发给我。”
4. After login: “已连上，正在检测系统版本、包管理器、Node.js 和 npm。”
5. Before install: “系统是 X，缺少 Y，我现在开始安装依赖。”
6. Before verification: “依赖已就绪，我现在安装 Codex，然后验证版本和 PATH。”

Bad behavior includes:

- asking the user to fill out a multi-field template
- asking the user whether Node.js is installed
- asking the user what Linux distribution they use before checking
- asking the user to send the SSH password in chat
- silently doing multiple steps without status updates

## Example Invocation Situations

Invoke this skill when the user says things like:

- “我要直接在云服务器上安装 codex。”
- “你一步步问我，然后帮我把 codex 装好。”
- “连到我的 VPS，别让我自己查环境，你自己装到能用为止。”
