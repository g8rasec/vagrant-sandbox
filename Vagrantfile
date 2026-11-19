# -*- mode: ruby -*-
# vi: set ft=ruby :

require "fileutils"

# Enable Vagrant's built-in disk management (no plugin required)
ENV["VAGRANT_EXPERIMENTAL"] = "disks"

# Per-machine overrides (gitignored), e.g.: DISK_SIZE = "200GB"
load "#{__dir__}/Vagrantfile.local" if File.exist?("#{__dir__}/Vagrantfile.local")

# ==============================================================================
# 1. VM CONFIGURATION CONSTANTS
# ==============================================================================
BOX_IMAGE        = "cloud-image/ubuntu-26.04" unless defined?(BOX_IMAGE)
PROJECT          = "dev-project" unless defined?(PROJECT)
CPUs             = 2 unless defined?(CPUs)
MEMORY           = "8192" unless defined?(MEMORY)
USERNAME         = "user" unless defined?(USERNAME)
PASSWORD         = "pass" unless defined?(PASSWORD)
SSH_KEY_FILENAME     = "id_ed25519" unless defined?(SSH_KEY_FILENAME)             # host key authorized to SSH INTO the VM
VM_GIT_PAT_FILENAME  = "github_pat_readonly" unless defined?(VM_GIT_PAT_FILENAME) # host file with a read-only GitHub fine-grained PAT, for HTTPS Git access inside the VM
DISK_SIZE            = "50GB" unless defined?(DISK_SIZE)
DOTFILES_REPO        = "https://github.com/your-username/dotfiles.git" unless defined?(DOTFILES_REPO)
# Dotfiles are optional: false skips them (a plain Ubuntu with the bootstrap packages and the AI CLIs).
USE_DOTFILES         = true unless defined?(USE_DOTFILES)
# Host checkout of the dotfiles (chezmoi's source dir). When it exists, the VM reads it through a read-only
# share and needs no GitHub token; otherwise it clones DOTFILES_REPO with the read-only PAT.
DOTFILES_SOURCE      = "~/.local/share/chezmoi" unless defined?(DOTFILES_SOURCE)
# Ubuntu archive mirror for the guest's apt. archive.ubuntu.com (the box default) can be
# unreachable while country mirrors still answer; "" keeps the box default.
# security.ubuntu.com is left as is.
APT_MIRROR           = "http://br.archive.ubuntu.com/ubuntu" unless defined?(APT_MIRROR)

# ==============================================================================
# 2. NETWORK CONFIGURATION
# ==============================================================================
# Network mode:
# - "private"       : Host-Only IP (always accessible via fixed IP from host)
# - "public_static" : Bridge with static IP
# - "public_dhcp"   : Bridge with DHCP (dynamic IP from router)
NETWORK_MODE             = "private" unless defined?(NETWORK_MODE)
VM_PRIVATE_IP            = "192.168.56.10" unless defined?(VM_PRIVATE_IP)
VM_BRIDGED_IP            = "192.168.15.10" unless defined?(VM_BRIDGED_IP)

# Interface prefix to auto-detect the host network card for bridged networking:
# - "en" or "eth"   : Wired Ethernet interfaces (e.g., enp1s0, eth0)
# - "wlp" or "wlan" : Wireless Wi-Fi interfaces (e.g., wlp0s20f3, wlan0)
NETWORK_INTERFACE_PREFIX = "en" unless defined?(NETWORK_INTERFACE_PREFIX)

# ==============================================================================
# 3. HELPER FUNCTIONS
# ==============================================================================
# Read SSH key safely without crashing if the file is missing on the host
def read_ssh_key(filename, is_public = false)
  path = File.expand_path("~/.ssh/#{filename}" + (is_public ? ".pub" : ""))
  if File.exist?(path)
    File.read(path).strip
  else
    puts "WARNING: Host SSH key not found at #{path}."
    ""
  end
end

# Detect the network interface matching the prefix
def detect_interface
  puts "Detecting network interface..."
  interface = `ip -o link show | awk -F': ' '{print $2}' | grep ^#{NETWORK_INTERFACE_PREFIX}`.strip
  if interface.empty?
    raise "Error: No network interface matching '#{NETWORK_INTERFACE_PREFIX}' was found."
  else
    puts "Detected network interface: #{interface}"
    interface
  end
end

# ==============================================================================
# 4. LOAD STATE & LOG CONFIGURATION
# ==============================================================================
# Derive names from the box's distro token, not its org (boxes are now
# "<org>/<distro>", e.g. "cloud-image/ubuntu-26.04"). Non-alnum -> "-".
BOX_TAG   = BOX_IMAGE.split("/").last.gsub(/[^A-Za-z0-9]+/, "-")
HOSTNAME  = "vm-" + BOX_TAG unless defined?(HOSTNAME)
VM_NAME   = ("vm-" + BOX_TAG + "-" + PROJECT).upcase

VM_SSH_PUB_KEY  = read_ssh_key(SSH_KEY_FILENAME, true)
VM_GIT_PAT      = read_ssh_key(VM_GIT_PAT_FILENAME, false)

DOTFILES_HOST_DIR = File.expand_path(DOTFILES_SOURCE)
DOTFILES_FROM_HOST = USE_DOTFILES && File.directory?(File.join(DOTFILES_HOST_DIR, ".git"))
DOTFILES_MOUNT = "/opt/dotfiles"
# Without the host's checkout, the repo is cloned only if one is set and a PAT can read it.
DOTFILES_CLONE = USE_DOTFILES && !DOTFILES_FROM_HOST && !VM_GIT_PAT.empty? && !DOTFILES_REPO.include?("your-username")
DOTFILES_MODE = if DOTFILES_FROM_HOST then "host checkout (#{DOTFILES_SOURCE}, read-only)"
                elsif DOTFILES_CLONE then "clone of #{DOTFILES_REPO} (PAT)"
                elsif USE_DOTFILES then "none found: skipped (see README, Dotfiles)"
                else "off (USE_DOTFILES = false)"
                end

# Push-capable SSH equivalent of DOTFILES_REPO, so the chezmoi source repo can be
# pushed to from inside the VM (via `ssh -A`) without carrying the read-only PAT
# over into its push URL. Only rewrites a plain "https://github.com/..." URL.
DOTFILES_REPO_SSH = DOTFILES_REPO.sub(%r{\Ahttps://github\.com/}, "git@github.com:")

GATEWAY_NETWORK = if NETWORK_MODE.start_with?("public")
                    `ip route | awk '/default/ && $5 ~ /#{NETWORK_INTERFACE_PREFIX}/ {print $3}'`.strip
                  else
                    "N/A (Private Network)"
                  end

# ==============================================================================
# 5. HOST TERMINFO SYNC
# ==============================================================================
# Captures the host's current $TERM terminfo entry so it can be installed on
# the guest. Without this, SSH clients using a TERM the guest doesn't know
# about (e.g. xterm-ghostty on older distros) get garbled/duplicated input,
# because cursor-control sequences from the shell prompt can't be resolved.
HOST_TERM = ENV["TERM"]
HOST_TERMINFO_PATH = "#{__dir__}/.vagrant/host-terminfo.terminfo"

if HOST_TERM
  FileUtils.mkdir_p("#{__dir__}/.vagrant")
  if system("infocmp -x #{HOST_TERM} > #{HOST_TERMINFO_PATH} 2>/dev/null")
    puts "Captured host TERM='#{HOST_TERM}' terminfo for guest sync."
  else
    HOST_TERMINFO_PATH = nil
    puts "Could not capture host TERM='#{HOST_TERM}' terminfo (infocmp unavailable or unknown to host too)."
  end
else
  HOST_TERMINFO_PATH = nil
end

# ==============================================================================
# 5.5 MACHINES (one VM, or one VM per project)
# ==============================================================================
# By default there is one VM, VM_NAME, configured by the constants above. To
# keep one VM per project, each started and stopped on its own, define MACHINES
# in Vagrantfile.local: the keys are the names used in commands
# (`vagrant up <name>`), and every option falls back to the constants above.
# See README "One VM per project".
#
#   MACHINES = {
#     "app-a" => { ip: "192.168.56.11", memory: "4096" },
#     "app-b" => { ip: "192.168.56.12", cpus: 4, hostname: "my-app-b" },
#   }
#
# Options: ip, hostname, display_name, cpus, memory, disk, autostart.
MACHINES = { VM_NAME => {} } unless defined?(MACHINES)
MULTI_MACHINE = !(MACHINES.size == 1 && MACHINES.key?(VM_NAME))

def machine_settings(name, opts)
  unknown = opts.keys - %i[ip hostname display_name cpus memory disk autostart]
  raise "MACHINES[#{name.inspect}]: unknown option(s) #{unknown.join(', ')}" unless unknown.empty?

  {
    ip:           opts.fetch(:ip, NETWORK_MODE == "public_static" ? VM_BRIDGED_IP : VM_PRIVATE_IP),
    # Hostnames can't hold "_" or ".", the names often can
    hostname:     opts.fetch(:hostname, MULTI_MACHINE ? name.downcase.gsub(/[^a-z0-9-]+/, "-") : HOSTNAME),
    display_name: opts.fetch(:display_name, defined?(VM_DISPLAY_NAME) && !MULTI_MACHINE ? VM_DISPLAY_NAME : nil),
    cpus:         opts.fetch(:cpus, CPUs),
    memory:       opts.fetch(:memory, MEMORY).to_s,
    disk:         opts.fetch(:disk, DISK_SIZE),
    # With several VMs, a bare `vagrant up` starts none: name the one you want
    autostart:    opts.fetch(:autostart, !MULTI_MACHINE),
  }
end

MACHINE_SETTINGS = MACHINES.to_h { |name, opts| [name, machine_settings(name, opts)] }

ips = MACHINE_SETTINGS.values.map { |m| m[:ip] }
if NETWORK_MODE != "public_dhcp" && ips.uniq.size != ips.size
  raise "Two machines share an IP (#{ips.tally.select { |_, n| n > 1 }.keys.join(', ')}): give each its own ip."
end

puts "=============================================================================="
puts "Configurations set:"
puts "  BOX_IMAGE:        #{BOX_IMAGE}"
puts "  USERNAME:         #{USERNAME}"
puts "  PASSWORD:         #{PASSWORD}"
puts "  NETWORK_MODE:     #{NETWORK_MODE}"
puts "  DOTFILES:         #{DOTFILES_MODE}"
puts "  APT_MIRROR:       #{APT_MIRROR.empty? ? "box default" : APT_MIRROR}"
if NETWORK_MODE.start_with?("public")
  puts "  INTERFACE_PREFIX: #{NETWORK_INTERFACE_PREFIX}"
  puts "  GATEWAY_NETWORK:  #{GATEWAY_NETWORK}"
end
MACHINE_SETTINGS.each do |name, m|
  ip = NETWORK_MODE == "public_dhcp" ? "DHCP (Dynamic)" : m[:ip]
  puts "  #{name}: #{ip}, hostname #{m[:hostname]}, #{m[:cpus]} CPUs, #{m[:memory]} MB, disk #{m[:disk]}" \
       "#{m[:autostart] ? '' : ', starts only by name'}"
end
puts "=============================================================================="


# ==============================================================================
# 6. VAGRANT CONFIGURATION
# ==============================================================================
Vagrant.configure("2") do |config|
  puts "Configuring Vagrant..."

  MACHINE_SETTINGS.each do |name, m|
    config.vm.define name, autostart: m[:autostart] do |host|
      puts "Defining VM: #{name}"
      host.vm.box      = BOX_IMAGE
      host.vm.hostname = m[:hostname]
      host.vm.disk :disk, size: m[:disk], primary: true

      # Share the host's ~/repos into the VM for repos cloned (and pushed) from the
      # host via SSH. Git network access for these repos never happens inside the
      # VM itself — only the resulting working tree is visible here.
      #
      # uid/gid=1000: placeholder only. Synced folders are mounted before the
      # shell provisioner creates USERNAME, so owner:/group: can't be used
      # (Vagrant would run `id -u` on a user that doesn't exist yet) and the
      # real UID/GID can't be known here on the host. It also isn't reliably
      # 1000 on the guest: official Canonical boxes already have "vagrant"
      # (1000) and cloud-init's default "ubuntu" user (1001) before useradd
      # ever runs, so USERNAME can land on 1002 or higher. The provisioner
      # below detects the real UID/GID after creating USERNAME and remounts
      # this share with the correct values.
      host.vm.synced_folder "~/repos", "/home/#{USERNAME}/repos", create: true, mount_options: ["uid=1000", "gid=1000"]

    # The host's dotfiles checkout, read-only: chezmoi applies it without cloning (no token in the VM).
    # Mounted outside the home, which does not exist yet; the provisioner links it into place.
    if DOTFILES_FROM_HOST
      host.vm.synced_folder DOTFILES_HOST_DIR, DOTFILES_MOUNT, mount_options: ["ro", "dmode=755", "fmode=644"]
    end

      # Apply network configuration based on mode
      case NETWORK_MODE
      when "private"
        puts "Configuring private network (Host-Only) with IP: #{m[:ip]}"
        host.vm.network "private_network", ip: m[:ip]
      when "public_dhcp"
        bridge_iface = detect_interface
        puts "Configuring public network (Bridge) via DHCP on #{bridge_iface}..."
        host.vm.network "public_network", type: "dhcp", bridge: bridge_iface
      when "public_static"
        bridge_iface = detect_interface
        puts "Configuring public network (Bridge) via Static IP #{m[:ip]} on #{bridge_iface}..."
        host.vm.network "public_network", ip: m[:ip], bridge: bridge_iface
      else
        raise "Invalid NETWORK_MODE: '#{NETWORK_MODE}'. Choose 'private', 'public_static' or 'public_dhcp'."
      end

      host.vm.provider "virtualbox" do |vb|
        vb.name   = m[:display_name] if m[:display_name]
        vb.memory = m[:memory]
        vb.cpus   = m[:cpus]
      end
    end
  end

  # VirtualBox settings common to every machine
  config.vm.provider "virtualbox" do |vb|
    vb.customize ["modifyvm", :id, "--acpi", "on"]
    vb.customize ["modifyvm", :id, "--ioapic", "on"]
    # Resolve DNS via the host resolver so the VM follows the host VPN DNS
    vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
  end

  # Sync host terminfo (see section 5 above) so SSH from the host's terminal
  # doesn't get garbled input if the guest doesn't recognize the host's TERM.
  if HOST_TERMINFO_PATH
    config.vm.provision "file", source: HOST_TERMINFO_PATH, destination: "/tmp/host.terminfo"
  end

  # Shell Provisioning
  config.vm.provision "shell", inline: <<-SHELL
    echo "Starting shell provisioning..."

    # 1. SSH Password Authentication
    echo "Enabling password authentication for SSH..."
    if [ -f /etc/ssh/sshd_config.d/60-cloudimg-settings.conf ]; then
      sudo sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config.d/60-cloudimg-settings.conf
    else
      sudo sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/' /etc/ssh/sshd_config
      sudo sed -i 's/PasswordAuthentication no/PasswordAuthentication yes/' /etc/ssh/sshd_config
    fi
    sudo systemctl restart ssh    # 24.04+ uses ssh.service (sshd is only an alias)

    # 2. Bootstrap Package Installation
    # Only what's strictly required before the user exists and dotfiles can
    # take over: zsh (login shell), curl+ca-certificates (fetch chezmoi
    # itself), ncurses-term (terminfo entries for common terminal emulators).
    # Everything else lives in the dotfiles repo's run_once scripts.
    if [ -n "#{APT_MIRROR}" ] && [ -f /etc/apt/sources.list.d/ubuntu.sources ]; then
      echo "Using apt mirror #{APT_MIRROR}..."
      sudo sed -i -E 's#^URIs: http://([a-z]+\\.)?archive\\.ubuntu\\.com/ubuntu/?$#URIs: #{APT_MIRROR}#' /etc/apt/sources.list.d/ubuntu.sources
    fi
    echo "Installing bootstrap packages..."
    # Stop here if this fails: the user's login shell is zsh, and every later step
    # runs as that user, so going on only piles up "zsh: command not found" errors.
    if ! { sudo apt-get update && sudo apt-get install -y zsh curl ca-certificates ncurses-term; }; then
      echo "ERROR: bootstrap packages failed to install (apt mirror unreachable?)." >&2
      echo "Set APT_MIRROR in Vagrantfile.local to a reachable mirror and run: vagrant provision <name>" >&2
      exit 1
    fi

    # 3. User Creation
    if ! id #{USERNAME} &>/dev/null; then
      echo "Creating user #{USERNAME}..."
      sudo useradd -m -s /usr/bin/zsh -G sudo #{USERNAME}
      echo "#{USERNAME}:#{PASSWORD}" | sudo chpasswd

      # useradd -m finds /home/#{USERNAME} already existing (created by root
      # by Vagrant as the mountpoint for the repos synced folder, before this
      # provisioner runs) and refuses to change its ownership or copy skel
      # files into it. Fix the home dir's ownership here — NOT recursive,
      # since /home/#{USERNAME}/repos is a separate vboxsf mount, fixed up
      # on its own in step 3.1 below.
      sudo chown #{USERNAME}:#{USERNAME} /home/#{USERNAME}

      echo "Copying default profiles..."
      sudo -u #{USERNAME} cp /etc/skel/.bashrc /home/#{USERNAME}/.bashrc
      sudo -u #{USERNAME} cp /etc/skel/.profile /home/#{USERNAME}/.profile
    else
      echo "User #{USERNAME} already exists."
    fi

    # 3.1. Repos synced-folder ownership
    # Handled by the dedicated "fix-repos-ownership" provisioner further down,
    # which runs on EVERY boot (run: "always"), not just under `--provision`.
    # Vagrant re-applies the placeholder uid/gid=1000 from the synced_folder
    # mount_options on each `vagrant up`, so the fix has to re-run every time.

    # 3.5. Host Terminfo Installation
    # Installs the host's TERM terminfo (copied in by the "file" provisioner
    # above, see section 5) into #{USERNAME}'s own ~/.terminfo, so SSH
    # sessions from the host's terminal emulator (e.g. Ghostty) don't get
    # garbled/duplicated input from unresolved cursor-control sequences.
    if [ -f /tmp/host.terminfo ]; then
      echo "Installing host terminfo for #{USERNAME}..."
      sudo -u #{USERNAME} -i tic -x /tmp/host.terminfo
      rm -f /tmp/host.terminfo
    fi

    # 4. SSH Credentials Configuration
    # Setup public key
    if [ -n "#{VM_SSH_PUB_KEY}" ]; then
      echo "Setting up SSH key for #{USERNAME}..."
      mkdir -p /home/#{USERNAME}/.ssh
      if ! grep -qF "#{VM_SSH_PUB_KEY}" /home/#{USERNAME}/.ssh/authorized_keys 2>/dev/null; then
        echo "#{VM_SSH_PUB_KEY}" >> /home/#{USERNAME}/.ssh/authorized_keys
      fi
      chmod 700 /home/#{USERNAME}/.ssh
      chmod 600 /home/#{USERNAME}/.ssh/authorized_keys
      chown -R #{USERNAME}:#{USERNAME} /home/#{USERNAME}/.ssh
    fi

    # Optional read-only GitHub PAT for #{USERNAME} (HTTPS, covers any repo the token grants read access to)
    if [ -n "#{VM_GIT_PAT}" ]; then
      echo "Setting up read-only GitHub PAT for #{USERNAME}..."
      sudo -u #{USERNAME} -i git config --global credential.helper store
      echo "https://x-access-token:#{VM_GIT_PAT}@github.com" > /home/#{USERNAME}/.git-credentials
      chown #{USERNAME}:#{USERNAME} /home/#{USERNAME}/.git-credentials
      chmod 600 /home/#{USERNAME}/.git-credentials
    else
      echo "No GitHub PAT on the host: in-VM git over HTTPS stays anonymous (clone on the host instead)."
    fi

    # 5. Apply Dotfiles (chezmoi), optional
    if [ "#{DOTFILES_FROM_HOST || DOTFILES_CLONE}" != "true" ]; then
      echo "Dotfiles: #{DOTFILES_MODE}."
    else
      echo "Applying dotfiles via chezmoi..."

      # Grant #{USERNAME} passwordless sudo only for this step: the dotfiles repo's
      # run_once scripts call plain "sudo apt-get install" internally, and this whole
      # provisioner runs non-interactively with no TTY, so sudo can't prompt for a
      # password here. Revoked immediately after chezmoi finishes, below — every other
      # step (AI CLI installs, interactive `vagrant ssh` sessions) never needs this.
      echo "#{USERNAME} ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/90-#{USERNAME}-nopasswd > /dev/null
      sudo chmod 440 /etc/sudoers.d/90-#{USERNAME}-nopasswd

      # The user's systemd instance, as after a login: dotfile scripts that enable user services
      # (systemctl --user) need it, and provisioning runs without a login session.
      sudo loginctl enable-linger #{USERNAME}
      USER_UID=$(id -u #{USERNAME})
      for _ in $(seq 1 30); do [ -S "/run/user/$USER_UID/bus" ] && break; sleep 1; done
      USER_BUS="XDG_RUNTIME_DIR=/run/user/$USER_UID DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/$USER_UID/bus"

      CHEZMOI_SRC="/home/#{USERNAME}/.local/share/chezmoi"
      if [ -d "#{DOTFILES_MOUNT}/.git" ]; then
        # The host's checkout, shared read-only: link it as chezmoi's source dir and apply. Updates come from a
        # `git pull` on the host, then `vagrant provision` (or `chezmoi apply` in the VM).
        echo "Applying dotfiles from the host's checkout (#{DOTFILES_SOURCE}, read-only)..."
        if [ -e "$CHEZMOI_SRC" ] && [ ! -L "$CHEZMOI_SRC" ]; then
          echo "Moving aside a chezmoi source dir cloned inside the VM..."
          mv "$CHEZMOI_SRC" "$CHEZMOI_SRC.cloned.$(date +%Y%m%d%H%M%S)"
        fi
        sudo -u #{USERNAME} mkdir -p "/home/#{USERNAME}/.local/share"
        sudo -u #{USERNAME} ln -sfn "#{DOTFILES_MOUNT}" "$CHEZMOI_SRC"
        if [ ! -x "/home/#{USERNAME}/.local/bin/chezmoi" ]; then
          sudo -u #{USERNAME} -i bash -c 'sh -c "$(curl -fsLS https://get.chezmoi.io)" -- -b ~/.local/bin'
        fi
        # --force: this runs with no TTY (see below); the dotfiles state always wins on this disposable VM.
        sudo -u #{USERNAME} -i env $USER_BUS bash -c '~/.local/bin/chezmoi init --apply --force'
      else
        # A previously failed clone (e.g. bad auth) can leave an incomplete, non-git source dir behind
        if [ -d "$CHEZMOI_SRC" ] && [ ! -d "$CHEZMOI_SRC/.git" ]; then
          echo "Removing incomplete chezmoi source directory from a previous failed attempt..."
          rm -rf "$CHEZMOI_SRC"
        fi

        if [ -d "$CHEZMOI_SRC/.git" ]; then
          echo "chezmoi source already present, pulling latest dotfiles..."
          # --force: this runs with no TTY, so if a managed file (e.g. .zshrc) was
          # touched outside chezmoi since the last apply — the AI CLI installers below
          # append PATH lines to it — chezmoi would otherwise try to prompt on /dev/tty,
          # fail to open it, and abort before running the run_once_after scripts. Since
          # this VM is disposable/reproducible, the dotfiles state should always win.
          sudo -u #{USERNAME} -i env $USER_BUS bash -c '~/.local/bin/chezmoi update --apply --force'
        else
          echo "Installing chezmoi and applying #{DOTFILES_REPO}..."
          sudo -u #{USERNAME} -i env $USER_BUS bash -c 'sh -c "$(curl -fsLS https://get.chezmoi.io)" -- -b ~/.local/bin init --apply #{DOTFILES_REPO}'
        fi

        # Point the chezmoi source repo's push URL at SSH instead of the read-only PAT's
        # HTTPS remote, so a `git push` from inside the VM (e.g. over `ssh -A`) has
        # somewhere to succeed, without ever making the read-only PAT itself push-capable.
        sudo -u #{USERNAME} -i git -C "$CHEZMOI_SRC" remote set-url --push origin #{DOTFILES_REPO_SSH}
      fi

      # Revoke passwordless sudo now that chezmoi (and any apt installs it triggered) is done.
      echo "Revoking passwordless sudo for #{USERNAME}..."
      sudo rm -f /etc/sudoers.d/90-#{USERNAME}-nopasswd
    fi

    # 6. AI CLI Tools (VM-only, not part of dotfiles)
    if ! sudo -u #{USERNAME} -i command -v agy &>/dev/null; then
      echo "Installing Antigravity CLI (agy) for #{USERNAME}..."
      sudo -u #{USERNAME} -i bash -c "curl -fsSL https://antigravity.google/cli/install.sh | bash"
    else
      echo "Antigravity CLI (agy) is already installed."
    fi

    if ! sudo -u #{USERNAME} -i command -v claude &>/dev/null; then
      echo "Installing Claude Code for #{USERNAME}..."
      sudo -u #{USERNAME} -i bash -c "curl -fsSL https://claude.ai/install.sh | bash"
    else
      echo "Claude Code is already installed."
    fi

    if ! sudo -u #{USERNAME} -i command -v codex &>/dev/null; then
      echo "Installing Codex CLI for #{USERNAME}..."
      sudo -u #{USERNAME} -i bash -c "curl -fsSL https://chatgpt.com/codex/install.sh | sh"
    else
      echo "Codex CLI is already installed."
    fi

  SHELL

  # Repos synced-folder ownership fix — runs on EVERY boot, not just --provision.
  #
  # The synced_folder mount_options above use a static placeholder uid/gid=1000,
  # since Vagrant mounts the share before USERNAME exists and the real UID/GID
  # depends on which regular users the box already ships with (e.g. "vagrant",
  # and cloud-init's default "ubuntu" user on official Canonical images) — so it
  # isn't reliably 1000. Vagrant re-applies that placeholder on every `vagrant
  # up`, which would lock USERNAME out of ~/repos until the next `--provision`.
  # run: "always" makes a plain `vagrant up` enough.
  config.vm.provision "fix-repos-ownership", type: "shell", run: "always", inline: <<-SHELL
    # Nothing to do until step 3 has created USERNAME. On first boot this
    # provisioner runs after the main one, so the user already exists by then.
    id #{USERNAME} &>/dev/null || exit 0

    REPOS_MOUNT="/home/#{USERNAME}/repos"
    REAL_UID=$(id -u #{USERNAME})
    REAL_GID=$(id -g #{USERNAME})
    # vboxsf never surfaces uid=/gid= in /proc/mounts (findmnt -o OPTIONS comes
    # back empty for them), so read the mapped owner off the mount point itself.
    CURRENT_UID=$(stat -c '%u' "$REPOS_MOUNT" 2>/dev/null || true)
    if [ -n "$CURRENT_UID" ] && [ "$CURRENT_UID" != "$REAL_UID" ]; then
      SHARE_NAME=$(awk -v mnt="$REPOS_MOUNT" '$2 == mnt {print $1}' /etc/fstab)
      echo "Remounting $REPOS_MOUNT with uid=$REAL_UID,gid=$REAL_GID (was uid=$CURRENT_UID) for #{USERNAME}..."
      # umount can fail with "target is busy" (e.g. a shell with its cwd inside
      # $REPOS_MOUNT). Only rewrite fstab if the remount actually succeeded —
      # otherwise fstab and the live mount end up out of sync, silently leaving
      # #{USERNAME} locked out of $REPOS_MOUNT.
      if sudo umount "$REPOS_MOUNT" && sudo mount -t vboxsf -o uid=$REAL_UID,gid=$REAL_GID "$SHARE_NAME" "$REPOS_MOUNT"; then
        sudo sed -i -E "\\|$REPOS_MOUNT|s/uid=[0-9]+,gid=[0-9]+/uid=$REAL_UID,gid=$REAL_GID/" /etc/fstab
      else
        echo "WARNING: failed to remount $REPOS_MOUNT (target busy?) — leave any shell whose cwd is inside it and re-run 'vagrant provision --provision-with fix-repos-ownership'." >&2
      fi
    fi
  SHELL
end

puts "Vagrant configuration completed."
