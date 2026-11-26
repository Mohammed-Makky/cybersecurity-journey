# cybersecurity-journey
I am Mohammed Musa Elfadil. A Sudanese BOIT student first year. Here I am trying to push myself as hard as I can from the comfort zone to start my cybersecurity journey. 
The first thing i did is creating an account in GitHub to track my progress 

##progress 
- [x] Created GitHub account and repository
- [ ] Project 1: Build the lab (Kali + Ubuntu Server)
- [ ] Project 2: Read the wire (Wireshark)
- [ ] Project 3: Python security tools
So fare, I will explain everything that happened step by step so, keep up with me:

1. First checks on Kali
I pooted Kali and ran three basic commands:
- (whoami) to shows which user I am.
- (ip a) to show network cards and IP addresses, which is VirtualBox's default address in NAT mode.
- (Ping -c 4 google.com) to check my internet connection.

2. The first problem that I faced: the disk filled up during the upgrade
I started a full system update with (sudo apt update && sudo apt full-upgrade -y). A "Low Disk Space" pop-up appeared saying only 691.8 MB was left.
- Cause: 2,266 packages needed upgrading (my Kali image was too old) and the downloads alone were several GB. My virtual disk was only 25 GB.
- What I did: While apt was still on "Get:" lines (downloading, not installing), I pressed Ctrl + C. That is safe during the downloads. Then I ran (sudo apt clean) to delete partial downloads, and (df -h /) showed 24 CB size,  18 GB used, 4.0 GB free (83% full).
- What I learned from this is to check free space before a big upgrade, and never interrupt apt once it reaches (Unpacking).

3. Resizing the virtual disk from (25GB to 60GB)
My VirtualBox version has no disk-size tool in the menus I could find, so I used the command line in Windows (Command Prompt) with the VM powered off:
"C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" modifymedium disk
"C:\Users\<my-user>\VirtualBox VMs\KaliMmad\KaliMmad.vdi" --resize 61440
61440 is the size in MB (60GB). VirtualBox's details panel then showed ('my Kali name' Normal, 60.00GB).
Then I took a snapshot, snapshot is a save point i can roll bask to if something breaks.

4. Making Linux use the new space
A bigger virtual disk dose nothing until the partition and filesystem inside it grow too. (lsblk) showed:
sda     60G   disk
├─sda1  23.7G  /       (system)
├─sda2  1K             (container for swap)
└─sda5  1.3G   [SWAP]
The new 35GB sat after the swap partition, so (sda1) could not grow into it. My steps:
sudo swapoff -a                         # stop using swap
sudo apt install -y cloud-guest-utils   # gives me growpart
sudo parted /dev/sda rm 5               # delete swap partition
sudo parted /dev/sda rm 2               # delete its container
sudo growpart /dev/sda 1                # grow partition 1 to fill the disk
sudo resize2fs /dev/sda1                # grow the filesystem inside it
df -h /                                 # now 59G, 39G free

5. Replacing swap with a swap file
Swap is a overflow space Linux uses when RAM runs low. I deleted the old swap partition, so I made a 2GB swap file instead:
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile      # only root can read it (swap can contain memory fragments)
sudo mkswap /swapfile
sudo swapon /swapfile
Then I told the system to load it at every boot by editing (/etc/fstab), the  file that lists what gets mounted at startup. I backed it up first:
sudo cp /etc/fstab /etc/fstab.bak
sudo sed -i '/<old-swap-UUID>/d' /etc/fstab            # delete the old swap line
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab   # add the new one (run once only)
I tested with (sudo swapoff -a && sudo swapon -a) and (free -h) showed 2.0GB swap.

6. The second problem: the download crawled at 5 to 15 KB/s
The upgrade summary said 3,779 MB to download. Early on it ran about 12 MB/s, then dropped to a few KB/s near 83%.
- How I found the cause: In the (Get:) lines, the slow files all came from (mirror.aktkn.sg). The fast ones came from (kali.download). The same slow mirror also appeared in my very first attempt. My internet was fine, one server was slow.
- Why it happens: Kali's setting (http.kali.org) is a redirector that picks a mirror (download server) for you, and sometimes picks a bad one.
- Fix: I pinned the fast mirror in (/etc/apt/sources.list.d/kali.sources). My first guess, (/etc/apt/sources.list), doesn't exist on new Kali versions, so I listed (/etc/apt/sources.list.d/) and found (kali.sources).
- sudo cp /etc/apt/sources.list.d/kali.sources /etc/apt/sources.list.d/kali.sources.bak
sudo sed -i 's|http://http.kali.org/kali/|http://kali.download/kali/|' /etc/apt/sources.list.d/kali.sources
sudo apt update
sudo apt full-upgrade
The download size then showed 693 MB instead of 3,779 MB (the rest was already on disk), at about 3 MB/s.

7. The third problem: black screen, then a half-finished upgrade
After about 45 minutes the VM window went black and did not react to keys. VirtualBox's Resources view showed low CPU and 0B/0B RAM reporting.
- I shut it down with VirtualBox's Machine>Stop>ACPI Shutdown (a polite "please shut down" signal) and started it again. Kali reached the desktop.
- I ran (sudo dpkg --configure -a) to finish half-configured packages. It stalled and printed (Reload daemon failed: Connection timed out), then (Transport endpoint is not connected). That meant systemd (the service manager) had stopped answering, probably because it was being upgraded in the middle of the run.
- I restarted the VM again and re-ran (dpkg --configure -a). The screen bounced back to the login screen (the desktop packages restart it when they are set up), but the terminal kept going.
- One package, (kali-desktop-gnome), failed because (gst-video-thumbnailer) had never been installed. I fixed that with (sudo --fix-broken install), which installed 3 small packages and removed nothing. Then (sudo apt full-upgrade) finished the remaining packages.
- Final check: (sudo dpkg --audit) printed nothing, then I rebooted into the new kernel.
Lesson: apt and dpkg can resume after interruptions. Fix with (dpkg --configure -a), then (apt --fix-broken install), then (apt full-upgrade). 

8. A leftover warning from deleting the swap partition:
   Every time the boot image was rebuilt I saw (cryptsetup: ERROR: Couldn't resolve device UUID=...) and (no matching swap device is available).
   - Cause: The file (/etc/initramfs-tools/conf.d/resume) still pointed to the swap partition I deleted.
   - Fix:
   echo "RESUME=none" | sudo tee /etc/initramfs-tools/conf.d/resume
sudo update-initramfs -u
My first attempt wrote to wrong place (/etc/initramfs-tools/conf), so I found the stray file with (grep -rn RESUME /etc/initramfs-tools/) and removed it with (sudo rm). After the correct fix the warnings were gone.

9. The fourth problem: Kali lost its internet (eth0).
After the upgrade, "ip a" showed "eth0" with no IP address, and "ping google.com" failed with "Temporary failure in name resolution".
- "nmcli device status" showed "eth0" as disconnected with on connection profile.
- The only profile, "Wired connection 1", belonged to "eth1".
- Fix: I created a profile for "eth0". NetworkManager uses automatic DHCP (the router-style service that hands out IP addresses) by default.
sudo nmcli connection add type ethernet ifname eth0 con-name nat-eth0
sudo nmcli connection up nat-eth0
"eth0" got "IP address" again and the ping worked.

10. Adding the privet lab network:
In VirtualBox (Settings > Expert > Network > Adapter 2) I enabled a second card set to Host-only Adapter. The Basic view only showed NAT and Bridged, so I switched to Expert mode. Kali now has:
- "eht0" on NAT (IP address) for internet and updates.
- "eth1" no host-only (IP address) for the lab network.
Before practicing attacks later I will untick Enable Network Adapter on Adapter 1, so the lab has no internet path.

Mini glossary (so I remember later) 
- VM: a computer running inside my computer as a window.
- NAT: the VM shares my laptop's internet but cannot be reached from outside.
- Host-only: a private network shared only by my laptop and my VMs.
- Partition/filesystem: a section of the disk, and the structure inside it that stores files. Both must be resized.
- Swap: disk space used as emergency memory when RAM is full.
- Mirror: one of many servers that host Kali's software downloads.
- apt / dpkg: the tools that download and install software. "apt" fetches, "dpkg" installs.
- Snapshot: a save point for a VM, so I can roll back after a mistake.
- systemd: the service manager that starts and tracks background services.
- sudo: run one command as administrator.

What I learned:
1. Check free disk space before a big upgrade (df -h /)
2. Read the logs. The slow mirror's name was visible the whole time.
3. Take a snapshot before risky changes, and keep backups of config files (.bak).
4. Troubleshoot one thing at a time, and write down the exact error text.
5. Typos are normal. Compare what I typed with what I was given, character by character.

##Progress
- [x] Create GitHub account and repository.
- [x] Kali VM running and fully upgraded
- [x] Virtual disk resized to 60 GB, swap rebuilt
- [x] Take snapshot "Kali-upgraded-clean"
- [] Create Ubuntu Server VM (the target)
- [] Ping test between Kali and Ubuntu over host-only network
- [] Add a network diagram and the lab IP addresses to this repo 
