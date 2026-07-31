# oo CLI Setup and Recovery

Read this file only when the `oo` CLI is missing or an `oo` command fails due
to authentication or configuration. The CLI is mandatory for the
video-subtitle workflow; do not continue with a different ASR or LLM provider.

## CLI Missing

When the shell reports `oo: command not found`, explain that the skill requires
the OOMOL `oo` CLI. Installing software changes the user's environment, so ask
for approval before running an installer. If the user prefers to install it
manually, provide the matching command.

macOS or Linux:

```bash
curl -fsSL https://cli.oomol.com/install.sh | bash
```

Windows PowerShell:

```powershell
irm https://cli.oomol.com/install.ps1 | iex
```

For other installation options, direct the user to:
<https://cli.oomol.com/install-guide.md>

After installation, open a new terminal or refresh `PATH`, then verify:

```bash
oo --version
```

Do not proceed until this succeeds. If the installer succeeds but the command
is still missing, inspect the installer's reported binary directory and add
that directory to `PATH`; do not guess a location or overwrite shell startup
files without approval.

## Authentication Missing

If an `oo` command reports that the user is not signed in, run or ask the user
to run:

```bash
oo auth login
```

Authentication may require the user to complete an interactive browser step.
Never ask the user to paste a password, session cookie, API key, or access token
into the conversation. After sign-in, retry the exact command that originally
failed.

## Capability or Account Error

Do not reinstall the CLI when the executable and authentication already work.
Report the original error when Fusion API access, account configuration, or
billing is unavailable, and preserve completed local files so the workflow can
resume later.

If the user declines installation or sign-in, stop. State that media upload,
Fusion API ASR, and OO-hosted translation are required parts of this skill, so
the skill cannot complete the requested workflow without `oo`.
