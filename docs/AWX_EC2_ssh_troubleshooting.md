# Fix AWX ⇄ Amazon Linux EC2 SSH (`Permission denied (publickey)`)

This error means the TCP connection to port 22 works, but the **SSH private key**
ansible uses does **not** match the instance’s **`authorized_keys`** (wrong key,
lost PEM, wrong host, etc.). Ansible code in this repo cannot “repair” SSH keys on
behalf of your account—you fix credentials and/or AWS access.

## 1. Confirm what key the instance expects

1. In **AWS Console** → **EC2** → select the instance (e.g. `52.x.x.x`).
2. Open **Details** (or **Connect**).
3. Note **Key pair name** (must match one key you control).

## 2. Put that key’s PEM in the AWX Machine credential

In **Automation Controller**:

1. **Credential** `pavan-aws-machine` → **SSH Private Key** = full PEM for that **same**
   key pair (including `-----BEGIN … KEY-----` and `-----END … KEY-----`).
2. **Username**: `ec2-user` (Amazon Linux 2 / AL2023), or omit if the JT supplies
   `-u ec2-user` (your template already does).
3. Leave **incorrect** keys/passwords unset so only the PEM is offered.
4. Ensure **Job template** attaches **both**:
   - **Machine** credential (`pavan-aws-machine`)
   - **AWS** credential (for boto3/API playbooks—not for SSH).

## 3. Sanity check outside AWX

From a machine where you have the PEM:

```bash
chmod 600 ~/keys/your-instance.pem
ssh -v -o IdentitiesOnly=yes -i ~/keys/your-instance.pem ec2-user@PUBLIC_IP_OR_DNS
```

If this fails locally, AWX cannot succeed until keys/host/key pair alignment is fixed.

## 4. If you lost the PEM (provision job created key once)

When `amazon.aws.ec2_key` **creates** a key pair, AWS returns the PEM **once**.
If the deploy job ran only in AWX you may never have persisted it—you **cannot**
re-download the private key.

**Practical outs:**

- Launch a replacement instance selecting a known key whose PEM you store inside
  an AWX **Machine** credential.
- Or recover access via **AWS support paths** suitable for your org (e.g. mount
  root volume offline, automation with **EC2 Instance Connect** + IAM, **SSM
  Session Manager** if agent + role are configured—then switch back to SSH if
  desired).

Keep new `.pem` in a **credential** or vault, never only in ephemeral job dirs.

## 5. Security group reminder (different symptom)

- **Timeouts / no route**: check **VPC**, **routing**, **NACL**, **security group**
  **inbound 22 from the Ansible execution subnet/IP**.
- **`Permission denied (publickey)`** usually means SG is OK and SSH reached
  `sshd`.

## 6. Job template: privilege escalation

After SSH works, this repo’s nginx role uses **become** for `dnf`/nginx:
ensure **Privilege escalation** is enabled with **Become method** “sudo”. Amazon
Linux `ec2-user` typically needs no password (`NOPASSWD`); leave **Become
password** empty unless yours is locked down.

---

After step 2–3 work, rerun **JT** `pavan-aws-deploywebapp` (playbook
`nginx_snake_demo.yml`). Also allow **TCP 80** in the SG for the Snake demo HTTP
traffic.
