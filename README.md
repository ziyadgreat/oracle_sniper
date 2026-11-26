# OCI Free ARM Instance Sniper

Oracle Cloud's **Always Free ARM instance** (`VM.Standard.A1.Flex`, up to 2 OCPU / 12 GB RAM)
is often unavailable in popular regions such as Mumbai, Hyderabad, Frankfurt or Ashburn,
failing with **"Out of host capacity"**.

This project runs on **GitHub Actions** every 15 minutes, retries the launch about once a
minute, and as soon as Oracle has capacity it creates the instance and **emails you** the
details. After success the workflow **disables itself**.

- Free: GitHub Actions minutes are unlimited for public repositories
- No PC or server has to stay on
- Safe in a public repo: no OCIDs, IPs, emails or keys are ever printed to the logs

---

## How it works

1. Checks whether an instance with the same name already exists (if so, it stops).
2. Auto-detects the availability domain, a public subnet and the latest Ubuntu 24.04 ARM image.
3. Tries the configured sizes in order, by default `2 OCPU/12 GB` → `1/6`.
4. On "out of capacity" it waits 60 s and retries; on `429 TooManyRequests` it backs off 3x longer.
5. On success it waits until the instance is `RUNNING`, sends the public IP and OCID **by email
   only**, and disables the workflow.

---

## Setup

### 1. Get your own copy
**Fork** this repository (or copy the files into a new repository). On a fork, open the
**Actions** tab and click *"I understand my workflows, go ahead and enable them"*.

> Keep the repository **public** so Actions minutes stay free. Secrets remain encrypted and
> hidden in public repositories. For private repositories see [Private repositories](#private-repositories).

### 2. Create an OCI API key
1. OCI Console → profile icon (top right) → **My profile** → **API keys** → **Add API key**
2. Select **Generate API key pair**, click **Download private key** (`.pem`), then **Add**
3. From the *Configuration file preview*, note `user`, `fingerprint`, `tenancy` and `region`

### 3. Create a network
OCI Console → **Networking → Virtual Cloud Networks → Start VCN Wizard →
Create VCN with Internet Connectivity**. The script finds the public subnet automatically.

### 4. Create an SSH key (works on Windows PowerShell, Linux and macOS)
```bash
ssh-keygen -t rsa -b 4096 -f oci_ssh
```
- Contents of `oci_ssh.pub` → secret `SSH_PUBLIC_KEY`
- Keep `oci_ssh` (the private key) safe on your machine; you log in with it. **Never commit it.**

### 5. Create an email app password (Gmail example)
1. Turn on **2-Step Verification** for the Google account
2. Create an app password at <https://myaccount.google.com/apppasswords>

> Consider a separate email account just for these notifications. Any SMTP provider works;
> set `SMTP_HOST` / `SMTP_PORT` accordingly (port 465 = SSL, anything else = STARTTLS).

### 6. Add the GitHub secrets
Repository → **Settings → Secrets and variables → Actions → New repository secret**

| Secret | Value | Required |
|---|---|---|
| `OCI_USER_OCID` | `user` from the configuration preview | Yes |
| `OCI_TENANCY_OCID` | `tenancy` from the configuration preview | Yes |
| `OCI_FINGERPRINT` | `fingerprint` from the configuration preview | Yes |
| `OCI_PRIVATE_KEY` | full contents of the downloaded `.pem`, including the BEGIN/END lines | Yes |
| `SSH_PUBLIC_KEY` | contents of `oci_ssh.pub` | Yes |
| `SMTP_USER` | sender email address | For email |
| `SMTP_PASS` | app password | For email |
| `EMAIL_TO` | recipient (defaults to `SMTP_USER`) | No |
| `OCI_KEY_PASSPHRASE` | only if your API key has a passphrase | No |
| `OCI_COMPARTMENT_OCID` | defaults to the root compartment (tenancy) | No |
| `OCI_SUBNET_OCID`, `OCI_IMAGE_OCID`, `OCI_AVAILABILITY_DOMAIN` | empty = auto-detect | No |

Optional **Variables** (same page, *Variables* tab). These are not secret and are visible in logs:

| Variable | Default | Meaning |
|---|---|---|
| `OCI_REGION` | `ap-mumbai-1` | your home region, e.g. `ap-hyderabad-1`, `eu-frankfurt-1` |
| `SHAPE` | `VM.Standard.A1.Flex` | instance shape; `VM.Standard.E2.1.Micro` (free AMD) is handy for a test run |
| `SHAPE_CONFIGS` | `2:12,1:6` | `OCPU:RAM_GB` sizes, tried in order (Flex shapes only) |
| `BOOT_VOLUME_GB` | `100` | boot disk size (Always Free total is 200 GB) |
| `DISPLAY_NAME` | `free-arm-server` | instance name |
| `RUN_MINUTES` | `25` | how long one run keeps retrying (keep below the workflow `timeout-minutes: 35`) |
| `RETRY_INTERVAL_SECONDS` | `60` | delay between attempts (don't go lower) |

### 7. Run it
**Actions → OCI A1 Sniper → Run workflow.** The log looks like this:
```
[10:01:12] Using image: Canonical-Ubuntu-24.04-aarch64-...
[10:01:13] Region ap-mumbai-1 | VM.Standard.A1.Flex | trying: 2 OCPU/12 GB, 1 OCPU/6 GB
[10:01:14] #1 2 OCPU/12 GB: out of capacity
```
After that the schedule keeps it running. When you get the email, connect with:
```bash
ssh -i oci_ssh ubuntu@<PUBLIC_IP>
```
(Oracle Linux images use the user `opc` instead of `ubuntu`.)

---

## Security

- **Never commit keys.** `.gitignore` already blocks `.pem`, `oci_ssh*`, `.oci/` and `.env`.
  Check `git status` before every commit.
- **Actions logs of public repositories are public.** Every account-specific value is therefore
  a *secret* (secrets are masked in logs; variables are not), and the script strips OCIDs, IPs,
  emails, fingerprints, URLs and keys from every line it prints. Instance details go only to email.
- **Forks never receive your secrets.** The workflow has no `pull_request` trigger, and scheduled
  workflows are disabled on forks by default.
- **If something leaks:** delete the API key in the OCI Console and create a new one, revoke the
  email app password, and update the GitHub secrets. Removing it from git history is not enough;
  the credential itself must be revoked.
- **Hide your personal email in commits:** GitHub → Settings → Emails → *Keep my email addresses
  private*, then use your `@users.noreply.github.com` address in `git config user.email`.

---

## Tips

- **Upgrading to Pay As You Go** is widely reported to make A1 capacity much easier to get. You are
  not charged while you stay within the Always Free limits (A1: 2 OCPU / 12 GB total since mid-2026; 200 GB block
  storage), but set up a **budget alert** (Billing → Budgets) just in case.
- If you only got a smaller size such as `1:6`, you can create a second instance with the remaining
  free quota: change `DISPLAY_NAME` and re-enable the workflow.
- GitHub disables scheduled workflows after 60 days without repository activity. Re-enable it from
  the Actions tab if needed.
- To run again later: **Actions → OCI A1 Sniper → Enable workflow**.

### Private repositories
Private repositories get 2,000 free Actions minutes per month. Set the variable `RUN_MINUTES=1`
and change the cron in `.github/workflows/oci-sniper.yml` to `*/30 * * * *`, otherwise the
minutes run out quickly.

## Running locally or on a Linux server
```bash
pip install -r requirements.txt
export OCI_USER_OCID=... OCI_TENANCY_OCID=... OCI_FINGERPRINT=...
export OCI_PRIVATE_KEY="$(cat ~/.oci/oci_api_key.pem)"
export SSH_PUBLIC_KEY="$(cat oci_ssh.pub)"
# A large RUN_MINUTES makes the script loop until it gets an instance
RUN_MINUTES=100000 python oracle_sniper.py
```

---

## Disclaimer
This project only uses Oracle's official API. You are responsible for complying with the Oracle
Cloud and GitHub terms of service for your own accounts.



