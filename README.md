# Fedora Linux setup notes by Blair

## Basic Information
- Last updated for Fedora 33

## Get the latest updates and get useful packages
```zsh
sudo dnf upgrade -y
sudo dnf install -y screen gnome-do audacity gimp redshift geany geany-themes git aria2
```
Some notes:
- Use command-line `redshift` because `redshift-gtk` doesn't seem to do what it is supposed to (no tray icon, runs in background)

## Git / GitHub credential saver
As per https://unix.stackexchange.com/questions/379272/storing-username-and-password-in-git
```zsh
git config credential.helper store
```

## Install some software manually (Dropbox, NetBeans, Edge, VSC, Teams)
Do Dropbox first so that it can start fetching your files in the background.
- https://www.dropbox.com/install-linux
- https://computingforgeeks.com/how-to-install-netbeans-ide-on-fedora/
- https://www.microsoftedgeinsider.com/en-us/download?platform=linux-rpm
- https://code.visualstudio.com/docs/setup/linux
- https://linuxhint.com/install-microsoft-teams-fedora/

## Install Spotify via Flatpak
As per https://docs.fedoraproject.org/en-US/quick-docs/installing-spotify/ - I am using the Flatpak option because I found that `lpf-spotify-client` was not building on my machine.

```zsh
sudo dnf install -y flatpak
flatpak remote-add --if-not-exists flathub https://flathub.org/repo/flathub.flatpakrepo
sudo flatpak install flathub com.spotify.Client
```

## Enable rpmfusion for Handbrake, VLC
Based on https://www.fosslinux.com/969/install-handbrake-fedora-22.htm

```zsh
su -c 'dnf install https://download0.rpmfusion.org/free/fedora/rpmfusion-free-release-$(rpm -E %fedora).noarch.rpm https://download0.rpmfusion.org/nonfree/fedora/rpmfusion-nonfree-release-$(rpm -E %fedora).noarch.rpm'
sudo dnf install -y handbrake handbrake-gui
sudo dnf install -y vlc
```

## GUI configuration
- Additional themes (e.g. Fitts) can be installed in `~/.themes`
- In file browser, go to Edit &rarr; Preferences and set default view to list view.
- To remove unwanted bookmarks in the file browser, delete the subfolders from the home folder (e.g. "Movies") and then run `xdg-user-dirs-update`.

## Setup folder for custom fonts

```zsh
mkdir /home/b/.local/share/fonts
# then copy whatever fonts you want
fc-cache -fv
```

## Enable Swapfile and Hibernation

1. Complete these steps from http://ctrl.blog/entry/fedora-hibernate.html which are required:

	- The step involving `dracut`
	- The `/etc/systemd/sleep.conf` step, but uncommenting these 3 lines:
		```
		AllowHibernation=yes
		HibernateMode=platform shutdown
		HibernateState=disk
		```
2. Swap file setup as per below based on https://superuser.com/questions/1581885/btrfs-luks-swapfile-how-to-hibernate-on-swapfile/1613639#1613639

	- (Part 1) Create and configure a swap-file the same size as the system memory
		```zsh
		sudo touch /var/swapfile
		sudo chattr +C /var/swapfile
		sudo fallocate --length "$(grep MemTotal /proc/meminfo | awk '{print $2 * 1024}')" /var/swapfile
		sudo chmod 600 /var/swapfile
		sudo mkswap /var/swapfile
		sudo swapon /var/swapfile
		echo '/var/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
		```

	- (Part 1.5) Calculate offset. We need to use the `btrfs_map_physical.c` code, otherwise the calculation won't take into account the idiosyncrasies of the btrfs filesystem.
		```zsh
		mkdir -p ~/00blair/fedorahibernation
		cd ~/00blair/fedorahibernation
		wget https://raw.githubusercontent.com/osandov/osandov-linux/master/scripts/btrfs_map_physical.c
		gcc -O2 -o btrfs_map_physical btrfs_map_physical.c
		sudo ./btrfs_map_physical /var/swapfile | sed -n "2p" | awk "{print \$NF}" >/tmp/swap_physical_offset
		```

	- (Part 2) Set the resume kernel parameters

		```zsh
		SWAP_PHYSICAL_OFFSET=$(cat /tmp/swap_physical_offset)
		SWAP_OFFSET=$(echo "${SWAP_PHYSICAL_OFFSET} / $(getconf PAGESIZE)" | bc)
		SWAP_UUID=$(findmnt -no UUID -T /var/swapfile)
		RESUME_ARGS="resume=UUID=${SWAP_UUID} resume_offset=${SWAP_OFFSET}"
		echo "${RESUME_ARGS}"
		```

	- (Part 2.5) If the output looks correct, set the kernel parameters with grubby:
		```zsh
		sudo grubby --update-kernel=ALL --args="${RESUME_ARGS}"
		```

	- (Part 3) Disable systemd swap space check.
		```zsh
		sudo mkdir -p /etc/systemd/system/systemd-logind.service.d/
		cat <<-EOF | sudo tee /etc/systemd/system/systemd-logind.service.d/override.conf
		[Service]
		Environment=SYSTEMD_BYPASS_HIBERNATION_MEMORY_CHECK=1
		EOF
		sudo mkdir -p /etc/systemd/system/systemd-hibernate.service.d/
		cat <<-EOF | sudo tee /etc/systemd/system/systemd-hibernate.service.d/override.conf
		[Service]
		Environment=SYSTEMD_BYPASS_HIBERNATION_MEMORY_CHECK=1
		EOF
		```

	- (Part 4)  SELinux: allow systemd to access the swap-file.
		```zsh
		cd "$(mktemp -dt)"
		cat <<-EOF | tee systemd_swapfile.te
		module systemd_swapfile 1.0;

		require {
			type init_t;
			type swapfile_t;
			class file { open getattr read ioctl lock };
		}

		allow init_t swapfile_t:file { open getattr read ioctl lock };
		EOF
		checkmodule -M -m -o systemd_swapfile.mod systemd_swapfile.te
		semodule_package -o systemd_swapfile.pp -m systemd_swapfile.mod
		sudo semodule -i systemd_swapfile.pp
		cd -
		```

	- (Part 5) Reboot and then try to hibernate using `systemctl hibernate`


## Shellstarterkit equivalent
TODO: send this to https://github.com/blairw/shellstarterkit/

- Install the Cascadia font, set it in VSC

- Some Terminal commands:
```zsh
sudo dnf install -y zsh
touch ~/.zshrc
chsh -s /bin/zsh
```

- Then log out and log back in, then:

```zsh
sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
git clone https://github.com/romkatv/powerlevel10k.git $ZSH_CUSTOM/themes/powerlevel10k
curl https://raw.githubusercontent.com/blairw/shellstarterkit/master/resources/dot-p10k.zsh -o ~/.p10k.zsh
code ~/.zshrc
```

In `~/.zshrc`:
- `ZSH_THEME="powerlevel10k/powerlevel10k"`
- `plugins=(git virtualenv)`
- End of file: `[[ ! -f ~/.p10k.zsh ]] || source ~/.p10k.zsh`
```
