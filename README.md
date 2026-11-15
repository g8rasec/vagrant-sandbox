# Vagrant Sandbox

Automated, portable Vagrant setup for a disposable development VM. Works
across network environments (home, work, offline).

## Features

* **Network modes** — Host-Only (private) or Bridge (static / DHCP).
* **Idempotent provisioning** — skips already-installed tools.
* **SSH key injection** — host public key added to the VM for passwordless login.
* **Host terminfo sync** — installs the host `$TERM` in the VM (fixes Ghostty etc.).
* **Host DNS** — VM resolves through the host resolver (follows host VPN DNS).
* **Dotfiles via chezmoi (optional)** — applies the host's dotfiles checkout (read-only share, no token) or clones `DOTFILES_REPO`.
* **AI CLIs preinstalled** — Antigravity (`agy`), Claude Code (`claude`), Codex (`codex`).

## Prerequisites

**1. VirtualBox 7.1** ([Oracle apt repo](https://www.virtualbox.org/wiki/Linux_Downloads)):

```bash
wget -O- https://www.virtualbox.org/download/oracle_vbox_2016.asc | sudo gpg --dearmor --yes --output /usr/share/keyrings/oracle-virtualbox-2016.gpg
echo "deb [arch=amd64 signed-by=/usr/share/keyrings/oracle-virtualbox-2016.gpg] https://download.virtualbox.org/virtualbox/debian $(lsb_release -cs) contrib" | sudo tee /etc/apt/sources.list.d/virtualbox.list
sudo apt update && sudo apt install virtualbox-7.1
```

> **Secure Boot on?** `vboxdrv` won't load until its module-signing key is
> enrolled: `sudo /sbin/vboxconfig`, then
> `sudo mokutil --import /var/lib/shim-signed/mok/MOK.der` and reboot
> (confirm the key in the blue MOK Manager screen).

**2. Vagrant** ([HashiCorp apt repo](https://developer.hashicorp.com/vagrant/install)):

```bash
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor --yes --output /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install vagrant
```

**3. SSH key pair** at `~/.ssh/id_ed25519{,.pub}` (`ssh-keygen -t ed25519`). The
public half is injected into the VM.

> Copied a key from a drive/backup/cloud sync? Fix its modes first — they
> often land world-readable and OpenSSH then silently falls back to a
> password prompt:
> `chmod 700 ~/.ssh && chmod 600 ~/.ssh/id_ed25519 && chmod 644 ~/.ssh/id_ed25519.pub`

**4. Dotfiles (optional)** — your [chezmoi](https://www.chezmoi.io/) dotfiles,
applied on provisioning. Without them (or with `USE_DOTFILES = false` in
`Vagrantfile.local`), the VM is a plain Ubuntu with the bootstrap packages and
the AI CLIs: install what your project needs (Docker, Node...) yourself. To add
them later, set `USE_DOTFILES = true` and run `vagrant provision <name>`. Two
ways, the first preferred:

```mermaid
flowchart LR
  H["Host: ~/.local/share/chezmoi<br/>(your dotfiles checkout)"] -- "read-only share<br/>/opt/dotfiles" --> V["VM: chezmoi init --apply"]
  R["DOTFILES_REPO on GitHub"] -- "clone with a read-only PAT<br/>(only without the checkout)" --> V
```

* **From the host's checkout (no token).** If [chezmoi](https://www.chezmoi.io/)'s
  source dir exists on the host (`DOTFILES_SOURCE`, default
  `~/.local/share/chezmoi`, a git checkout), it is shared read-only into the VM
  and linked as the VM's chezmoi source. Nothing is cloned and no GitHub
  credential enters the VM. Get it once with `chezmoi init <your-dotfiles-repo>`
  on the host. To update the VM's dotfiles: `git pull` in the checkout on the
  host, then `vagrant provision <name>` (or `chezmoi apply` in the VM).
* **Cloned inside the VM (needs a token).** Without the checkout, the VM clones
  `DOTFILES_REPO` over HTTPS with a read-only PAT:
  1. Point `DOTFILES_REPO` at it in a gitignored `Vagrantfile.local`:
     ```ruby
     DOTFILES_REPO = "https://github.com/your-username/your-dotfiles.git"
     ```
  2. A fine-grained PAT (**Contents: Read-only**; resource owner = the repo's
     owner, so an organization's repo needs a token owned by that organization)
     saved plain:
     ```bash
     read -rs T && printf '%s' "$T" > ~/.ssh/github_pat_readonly && chmod 600 ~/.ssh/github_pat_readonly; unset T
     ```
  3. Verify it (expect `200`; `401` = bad token, `404` = no repo access):
     ```bash
     curl -s -o /dev/null -w "%{http_code}\n" \
       -H "Authorization: Bearer $(tr -d '\n' < ~/.ssh/github_pat_readonly)" \
       https://api.github.com/repos/your-username/your-dotfiles
     ```

  With a PAT present (either way), the VM also gets read-only HTTPS access to
  GitHub for in-VM `git clone`/`pull` — see [Repos & Git](#repos--git).

Either way, provisioning turns on the VM user's systemd instance
(`loginctl enable-linger`), as a login would, so dotfile scripts that enable
user services (`systemctl --user`) work; user services then keep running after
you log out of the VM.

**Undo.** The VM only reads the checkout; `vagrant destroy <name>` leaves the
host's dotfiles as they were. To stop sharing them, set `DOTFILES_SOURCE` to a
path that does not exist (the VM then needs the PAT).

## Network modes (`NETWORK_MODE`)

| Value | Mode | IP | Use |
| :--- | :--- | :--- | :--- |
| `"private"` | Host-Only (default) | `192.168.56.10` | Daily use; no IP conflicts, works offline. |
| `"public_static"` | Bridge, static | `192.168.15.10` | Other LAN devices need to reach the VM. |
| `"public_dhcp"` | Bridge, DHCP | from router | Public connection where IPs are dynamic. |

## Getting started

```bash
cd ~/repos/vagrant-sandbox
vagrant up
vagrant ssh                    # or: ssh user@192.168.56.10
vagrant halt | reload | provision
```

> `ssh` asking for a password instead of using the key = bad key permissions
> (see Prerequisites 3). Confirm with `ssh -v user@192.168.56.10`.

`~/.ssh/config` entry for `ssh vm-ubuntu`:

```ssh-config
Host vm-ubuntu
    HostName 192.168.56.10
    User user
    StrictHostKeyChecking accept-new
```

## One VM per project

Projects that each need the same ports (an ingress on `443`, a Kubernetes
cluster) can't share a VM: whichever starts last takes the port. Give each
project its own VM, with its own IP on the Host-Only network, and start only
the ones you are testing.

```mermaid
flowchart LR
  subgraph host["Host"]
    B["Browser / IDE"]
  end
  subgraph net["Host-Only network 192.168.56.0/24"]
    A["app-a VM<br/>192.168.56.11<br/>its own cluster, :443"]
    C["app-b VM<br/>192.168.56.12<br/>its own stack, :443"]
  end
  B -- "https://app-a.test" --> A
  B -- "https://app-b.test" --> C
```

Define `MACHINES` in `Vagrantfile.local`. The keys are the names used in
commands; every option falls back to the constants of the `Vagrantfile`:

```ruby
MACHINES = {
  "app-a" => { ip: "192.168.56.11", memory: "6144" },
  "app-b" => { ip: "192.168.56.12", memory: "4096", hostname: "my-app-b" },
}
```

| Option | Default |
| :--- | :--- |
| `ip` | `VM_PRIVATE_IP` (or `VM_BRIDGED_IP` in `"public_static"`); must be unique |
| `hostname` | the machine name |
| `display_name` | Vagrant default |
| `cpus` / `memory` / `disk` | `CPUs` / `MEMORY` / `DISK_SIZE` |
| `autostart` | `false`: a bare `vagrant up` starts nothing |

```bash
vagrant status                 # every VM and its state
vagrant up app-a               # creates or starts only app-a
vagrant halt app-a             # stops it, keeps its disk
vagrant ssh app-b              # or: ssh user@192.168.56.12
vagrant destroy app-b          # deletes that VM and its disk
```

A recreated VM has a new SSH host key, and `ssh` refuses it ("REMOTE HOST
IDENTIFICATION HAS CHANGED"). Drop the old one after a `destroy`:
`ssh-keygen -R 192.168.56.12` (its `ip`).

The host's RAM is shared by the running VMs: start the ones whose memory adds
up to what the host can spare. Each VM gets its own `~/.ssh/config` entry
(`HostName` = its `ip`).

**Keeping an existing VM.** Vagrant identifies a VM by the folder
`.vagrant/machines/<name>`. To keep the VM you already have as one of the
named machines, give it its current IP and names in `MACHINES` and rename the
folder while nothing runs a `vagrant` command:

```bash
mv .vagrant/machines/VM-UBUNTU-24-04-DEV-PROJECT .vagrant/machines/app-a
vagrant status                 # app-a: running (or poweroff)
```

**Back to a single VM.** `vagrant destroy` the VMs you no longer want, remove
`MACHINES` from `Vagrantfile.local` and rename the folder of the VM you keep
back to its `VM_NAME`
(`mv .vagrant/machines/app-a .vagrant/machines/VM-UBUNTU-24-04-DEV-PROJECT`).

## Customization (`Vagrantfile`)

Variables at the top of the `Vagrantfile` (override per machine in a
gitignored `Vagrantfile.local`):

| Variable | Default | Notes |
| :--- | :--- | :--- |
| `BOX_IMAGE` | `cloud-image/ubuntu-24.04` | Base box. Canonical stopped publishing `ubuntu/*` boxes after jammy; this is the cloud-image repack. |
| `PROJECT` | `dev-project` | Label mixed into `VM_NAME` (the single VM; see [One VM per project](#one-vm-per-project) for several). |
| `HOSTNAME` | `vm-<box distro>` | Guest hostname; override in `Vagrantfile.local` without changing `VM_NAME` or the existing VM identity. |
| `VM_DISPLAY_NAME` | Vagrant default | Name shown in VirtualBox; override in `Vagrantfile.local` without changing the existing VM identity. |
| `CPUs` / `MEMORY` | `2` / `8192` | Cores / RAM (MB). |
| `DISK_SIZE` | `50GB` | Grow on `vagrant reload`; shrink needs a rebuild. |
| `USERNAME` / `PASSWORD` | `user` / `pass` | VM user. |
| `SSH_KEY_FILENAME` | `id_ed25519` | Host `~/.ssh` public key to inject. |
| `VM_PRIVATE_IP` | `192.168.56.10` | IP for `"private"` mode. |
| `VM_BRIDGED_IP` | `192.168.15.10` | IP for `"public_static"` mode. |
| `NETWORK_INTERFACE_PREFIX` | `en` | Host NIC prefix for Bridge (`wlp` = Wi-Fi). |
| `DOTFILES_REPO` | placeholder | chezmoi dotfiles HTTPS URL. |
| `USE_DOTFILES` | `true` | `false` skips the dotfiles: a plain Ubuntu VM. |
| `DOTFILES_SOURCE` | `~/.local/share/chezmoi` | Host dotfiles checkout, shared read-only; when present, no PAT is needed. |
| `VM_GIT_PAT_FILENAME` | `github_pat_readonly` | Host `~/.ssh` PAT file; optional with the checkout, required without it. |

## Running `vagrant` from any directory

`vagrant` only finds this VM from a directory with a `Vagrantfile`. Add this
wrapper to `~/.zshrc` or `~/.bashrc` (works in both) to point it here by
default while still respecting other Vagrant projects:

```bash
vagrant() {
  if [[ -f Vagrantfile || -f ../Vagrantfile || -n "$VAGRANT_CWD" ]]; then
    command vagrant "$@"
  else
    VAGRANT_CWD="$HOME/repos/vagrant-sandbox" command vagrant "$@"
  fi
}
```

(A plain `export VAGRANT_CWD="$HOME/repos/vagrant-sandbox"` also works but
targets this sandbox from *everywhere*.) Re-`source` the rc file afterwards.

## Commit conventions

[Conventional Commits](https://www.conventionalcommits.org/pt-br/v1.0.0-beta.4/):

* Subject `<type>(<scope>)?: <desc>` — ≤ 50 recommended, **72 hard limit**.
  Types: `build chore ci docs feat fix perf refactor revert style test`.
  `!` before `:` marks a breaking change.
* Body optional: blank line after the subject, lines wrapped at **72**.
* No `Co-Authored-By:` trailer.

Enforced by a global `commit-msg` hook (via `core.hooksPath`) managed in this
maintainer's personal dotfiles, not tracked in this repo — no per-clone setup
needed here.

## Moving files to the VM

`shared/` is a drop folder between the host and every VM: put a file there on
the host and it shows up in the VM at `~/repos/vagrant-sandbox/shared` (through
the `~/repos` share), and the other way around.

The folder is versioned empty: everything inside it is gitignored, so nothing
dropped there reaches this public repo. Clear it when the transfer is done.

```bash
cp ~/Downloads/report.pdf ~/repos/vagrant-sandbox/shared/   # on the host
ls ~/repos/vagrant-sandbox/shared                            # in the VM
```

## Repos & Git

**Recommended — keep git on the host.** `~/repos` is shared into the VM at
`/home/USERNAME/repos`. Edit inside the VM; run all `clone`/`fetch`/`pull`/`push`
from the host with your own keys. The repo then survives `vagrant destroy`, and
a disposable VM never touches your remotes.

**Cloning inside the VM.** The VM's PAT is read-only HTTPS — `clone`/`pull` of
any covered repo works, but the copy dies with the VM and `push` is rejected.
To push from inside anyway: forward your agent (`ssh -A vm-ubuntu`) and set the
push URL to SSH (`git remote set-url --push origin git@github.com:you/repo.git`).

With the host's checkout, the VM's `~/.local/share/chezmoi` is a read-only
link to it: commit and push dotfile changes on the host. When the VM cloned
`DOTFILES_REPO` itself (PAT mode), its push URL is already SSH, so `ssh -A` +
`git push` in `~/.local/share/chezmoi` just works.
