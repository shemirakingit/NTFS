# NTFS File Server Lab — Google Cloud Platform (GCP)

An Active Directory, NTFS permissions and Group Policy lab built on **Google Cloud Platform** with **Terraform**. Three Windows Server 2022 VMs on Compute Engine form a small domain (`lab.local`): a domain controller, a file server with department shares, and a workstation where test users exercise the full permission chain.

The Windows Server side covers AD DS, OUs, security groups, NTFS permissions and Group Policy; the infrastructure underneath (networking, VMs, secrets, remote state, script delivery) is provisioned on GCP with Terraform.

**Platform:** Google Cloud Platform (Compute Engine, VPC, Secret Manager, Cloud Storage)
**Tooling:** Terraform (`google` and `random` providers), `gcloud` CLI, PowerShell

## Architecture

| VM | Role | Internal IP |
| --- | --- | --- |
| `dc01` | Domain controller: AD DS, DNS, Group Policy for `lab.local` | 10.0.1.4 |
| `fs01` | File server: four SMB shares with NTFS permissions per security group | 10.0.1.5 |
| `client01` | Workstation where the test users log in over RDP | 10.0.1.6 |

All three sit in one VPC subnet (`10.0.1.0/24`). A firewall rule allows RDP (3389) **only from your own IP**; all other traffic is limited to the lab subnet.

### GCP components

| Purpose | GCP service | Terraform resource |
| --- | --- | --- |
| Container for all resources | Project | `google_project` (usually pre-existing) |
| Network | VPC network + subnet | `google_compute_network`, `google_compute_subnetwork` |
| RDP restricted to your IP | Firewall rule | `google_compute_firewall` |
| Servers | Compute Engine instances | `google_compute_instance` |
| Admin password | Secret Manager | `google_secret_manager_secret`, `google_secret_manager_secret_version` |
| Remote Terraform state | Cloud Storage bucket | created with `gcloud storage` (see below) |
| Running scripts on the VMs | Script pushed through instance metadata, pulled and run inside the VM | `lab-script` metadata key |
| Fixed addresses | Static internal IPs | `network_ip` on the instance |

## Repository layout

```
.
├── README.md
├── providers.tf                # google + random providers
├── variables.tf                # project_id, region, zone, my_ip
├── terraform.tfvars.example    # copy to terraform.tfvars (git-ignored)
├── backend.tf                  # GCS remote state (set your bucket name)
├── network.tf                  # VPC, subnet, firewall rules
├── secrets.tf                  # generated admin password + Secret Manager + service account
├── vms.tf                      # dc01, fs01, client01
├── outputs.tf                  # external IPs, secret name
└── scripts/
    ├── set-admin-password.ps1  # boot-time startup script (set by Terraform)
    ├── 01-promote-dc.ps1       # optional scripted DC promotion (untested; lab used the GUI)
    ├── 02-ad-objects.ps1       # OUs, security groups, test users
    ├── 03-join-domain.ps1      # point DNS at DC01 and join lab.local
    ├── 04-shares.ps1           # SMB shares + NTFS permissions
    └── 05-test-access.ps1      # read/write test per share
```

## Prerequisites

- A GCP project with billing enabled. **A free trial account cannot create Windows VMs** (error `windowsVmNotAllowedInFreeTrialProject`). Upgrade the billing account to a paid one; trial credits are kept and applied first. Set a budget alert.
- `gcloud` CLI installed and authenticated:
  ```
  gcloud auth login
  gcloud auth application-default login
  gcloud config set project <your-project-id>
  ```
- Terraform installed locally.
- Your public IPv4 address: `curl -4 ifconfig.me`
- APIs enabled:
  ```
  gcloud services enable compute.googleapis.com secretmanager.googleapis.com storage.googleapis.com
  ```
- An RDP client. On a Mac, use Microsoft's **Windows App**; allow it under System Settings > Privacy & Security > Local Network.

## Deploy the infrastructure

**1. Create the remote state bucket** (names are global, so pick your own):

```
gcloud storage buckets create gs://<your-unique-bucket-name>-tfstate --location=us-central1
gcloud storage buckets update gs://<your-unique-bucket-name>-tfstate --versioning
```

**2. Set the bucket name** in `backend.tf`, and create your values file:

```
cp terraform.tfvars.example terraform.tfvars   # then edit project_id and my_ip
```

**3. Deploy:**

```
terraform init
terraform plan
terraform apply
```

Outputs list each VM's external IP and the secret name. Windows needs 5-10 minutes after the VMs are created to finish first boot and run the password startup script.

**4. Get the admin password** (the same password is used for every VM):

```
gcloud secrets versions access latest --secret="ntfs-lab-admin-password"
```

On a Mac, add `| pbcopy` to copy it instead of printing it.

## Configure Windows

Scripts are delivered by pushing them into a VM's metadata and pulling them down from inside the VM. It needs no WinRM and no extra open ports.

**Push** (Mac or Linux terminal, in this folder; one VM per command):

```
gcloud compute instances add-metadata <vm> --zone=us-central1-a --metadata-from-file=lab-script=scripts/<script>.ps1
```

**Pull and run** (elevated PowerShell on that VM; set `$name` to the script you pushed):

```powershell
$name = '04-shares.ps1'
$h = @{'Metadata-Flavor'='Google'}
$r = Invoke-WebRequest -UseBasicParsing -Headers $h -Uri http://169.254.169.254/computeMetadata/v1/instance/attributes/lab-script
$t = if ($r.Content -is [byte[]]) { [Text.Encoding]::UTF8.GetString($r.Content) } else { $r.Content }
New-Item -ItemType Directory -Path C:\lab -Force | Out-Null
Set-Content -Path "C:\lab\$name" -Value $t -Encoding UTF8
powershell -ExecutionPolicy Bypass -File "C:\lab\$name"
```

Always run `hostname` first to confirm which VM you are on.

### 1. Promote DC01

Log in to DC01 as `Administrator`. In Server Manager: **Manage > Add Roles and Features** > Active Directory Domain Services, then the yellow flag > **Promote this server to a domain controller** > new forest `lab.local` (NetBIOS `LAB`), set a DSRM password. DC01 restarts. Reconnect as `LAB\Administrator` and check:

```powershell
Get-ADDomain | Select-Object DNSRoot, NetBIOSName
Get-Service NTDS, DNS | Select-Object Name, Status
```

`scripts/01-promote-dc.ps1` is an untested scripted alternative.

### 2. OUs, groups and users (DC01)

Push `02-ad-objects.ps1` to `dc01`, then pull and run it on DC01. Creates OUs and security groups `SG_Finance`, `SG_HR`, `SG_IT`, `SG_Sales` and the five test users.

### 3. Join FS01 and CLIENT01 to the domain

Push `03-join-domain.ps1` to `fs01` and `client01` (separate commands), then pull and run on each. Each VM points its DNS at DC01, waits for `lab.local` to resolve, joins the domain and restarts. DC01's DNS forwards GCP's internal names to `169.254.169.254`; confirm with `Get-DnsServerForwarder` on DC01.

Verify on DC01: `Get-ADComputer -Filter * | Select-Object Name` should list all three machines.

### 4. File shares (FS01)

Push `04-shares.ps1` to `fs01`, then pull and run on FS01. Creates `C:\Shares\{Finance,HR,IT,Sales}` as SMB shares with NTFS permissions per group.

### 5. RDP access for the test users (GPO)

CLIENT01 is a server, so ordinary users cannot use Remote Desktop by default. On DC01 (elevated PowerShell):

```powershell
New-ADOrganizationalUnit -Name Workstations -Path "DC=lab,DC=local" -ProtectedFromAccidentalDeletion $false
Get-ADComputer CLIENT01 | Move-ADObject -TargetPath "OU=Workstations,DC=lab,DC=local"
```

Then in **Group Policy Management**, create a GPO named `RDP-Access` linked to the `Workstations` OU. Edit it: **Computer Configuration > Policies > Windows Settings > Security Settings > Restricted Groups**, add the group `Remote Desktop Users`, and add `LAB\SG_Finance`, `LAB\SG_HR`, `LAB\SG_IT`, `LAB\SG_Sales` as its members. On CLIENT01 run `gpupdate /force`, then `net localgroup "Remote Desktop Users"` to confirm the four groups are listed.

## Test the permissions

Sign out of the Administrator session on CLIENT01 first (Windows Server allows two remote sessions). Log in as each user (password for all five: `P@ssw0rd123!`) and run `scripts/05-test-access.ps1`.

| User | Department | Finance | HR | IT | Sales |
| --- | --- | --- | --- | --- | --- |
| `LAB\sarah` | Finance | read + write | denied | denied | denied |
| `LAB\mike` | Finance | read + write | denied | denied | denied |
| `LAB\lisa` | HR | read only | read + write | denied | denied |
| `LAB\john` | IT | read + write | read + write | read + write | read + write |
| `LAB\tom` | Sales | denied | denied | denied | read + write |

Result of the last full run: all five users matched the table.

## Cost control

Windows VMs bill while running. After every session:

```
gcloud compute instances stop dc01 fs01 client01 --zone=us-central1-a
```

Stopped VMs keep their disks (a small ongoing cost) and **get new external IPs when restarted**. Start `dc01` first, wait about 5 minutes, then start `fs01` and `client01`. Check the current IPs with:

```
gcloud compute instances list --format="table(name,status,networkInterfaces[0].accessConfigs[0].natIP)"
```

To remove everything: `terraform destroy`. The state bucket is not managed by Terraform and stays until you delete it.

If your own IP changes, update `my_ip` in `terraform.tfvars` and run `terraform apply`; the plan should show only the firewall rule changing.

## Rebuilding the lab

- **Stop and start** keeps the domain, shares and GPO. Nothing is rebuilt.
- **Destroy and re-apply** recreates the infrastructure and generates a new admin password. The Windows configuration is lost, so repeat the "Configure Windows" steps in order.

Always read `terraform plan` before applying. In-place changes (firewall, metadata) are safe; a plan that replaces a VM wipes its disk.

## Lessons learned and troubleshooting

- **Free trial blocks Windows VMs.** Upgrade the billing account (credits are kept) and set a budget alert. In the budget's scope, uncheck "Promotional credits" if you want alerts on real spend while credits pay the bill.
- **No Windows 11 image on Compute Engine.** Public images are Windows Server only, so CLIENT01 is Windows Server 2022 acting as the workstation.
- **Startup-script order.** Set the Administrator password *before* enabling the account, otherwise Windows rejects it with a misleading complexity error.
- **Metadata comes back as bytes** in Windows PowerShell 5.1; decode it before running (the pull block does this).
- **`gcloud compute instances add-metadata` takes one VM per command.**
- **`0x104` in Windows App** means it cannot find the PC: the tile's *PC name* must be the VM's current external IP (not its label), the IP changes after every stop/start, and Windows App needs Local Network access on macOS.
- **Login names after the domain exists:** try `LAB\Administrator`, `<VM>\Administrator`, then `Administrator`.
- **Fallback access:** `gcloud compute reset-windows-password <vm> --zone=us-central1-a --user=labadmin` creates a fresh local admin.
- **A trailing `%` in terminal output on a Mac** is zsh marking a missing newline, not part of the value.
- **Use `curl -4 ifconfig.me`**: the firewall rule needs an IPv4 address.

## Security notes

- RDP is open only to the IP in `my_ip`. Keep it that way, and keep the VMs stopped when unused.
- The generated admin password is stored in Secret Manager and also appears in Terraform state, so keep the state bucket private and never commit `*.tfstate` or `terraform.tfvars`.
- `P@ssw0rd123!` is a well-known lab password for the five test users only. Do not reuse it anywhere real.

## Possible next steps

- Script the `Workstations` OU and the `RDP-Access` GPO so a rebuild needs no GUI steps.
- Test or replace `01-promote-dc.ps1` so DC promotion is fully scripted.
- Add a verification script that checks NTFS permission flags with a bitwise AND.
