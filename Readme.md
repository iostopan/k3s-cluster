# Setting up k3s cluster running on Proxmox VE

## Intro and Base Setup
This is a documentation of my homelab project to learn virtualization, kubernetes, container deployment, nginx, database management systems and web security. It will be changing and evolving, mostly serving as a guide for myself but also to everyone who would like to try out the project.

### Hardware

The hardware I'll be using for this project is an old laptop of mine - `ASUS N56JR` with a few changed specifications.

>CPU: Intel Core i7-4700HQ (4-core, 2.40 - 3.40 GHz) / RAM: 12GB (1x 4096MB, 1x 8192MB) - DDR3, 1600Mhz / HDD: 256 GB

It's connected via cable to a router and accessed via SSH from my main NUC workstation via Wi-Fi.

### Server
The base of the server is [Proxmox VE 7.2](https://www.proxmox.com/en/). An open-source virtualization platform and hypervisor based on Debian 11. ISO was installed directly through the interactive installer on the laptop.

### Use
Access to the hypervisor web interface is through the given address at `192.168.X.X:8006`, using the login user and password set during installation. From here on, everything is done through the NUC workstation and the laptop is left closed and running.

The `NUC8i3BEH` workstation is currently running `Solus OS x86_64`. Desktop config files can be found in the [`desktop` repo](https://github.com/iostopan/desktop)


![[scrot.png]]

## Setting up VM Environment
For this step I've been consulting a tutorial on [EnigmaCurry's dev blog](https://blog.rymcg.tech/)

Most of the code is taken from there.

### SSH keys

1. Created SSH host entry in ``$HOME/.ssh/config`` file:


```bash
Host pve 
	Hostname 192.168.X.X 
	User root
```

Changing the `192.168.X.X` to the IP of the Proxmox virtual machine.

2. Created an SSH identity on the workstation running `ssh-keygen`
3. Run `ssh-copy-id pve` to confirm SSH key fingerprint and remote password chosen during install to login to the Proxmox server via SSH.
4. SSH to the Proxmox host: `ssh pve`
5. Disable password authentication editing with your favorite text editor `/etc/ssh/sshd_config`
6. Uncomment `#` on the PasswordAuthentication line and change `yes` to `no`
7. Save `/etc/ssh/sshd_config` and close the editor
8. Restart ssh, run: `systemctl restart sshd`
9. Testing if `PasswordAuthentication` is really turned off with a non-existant username: `ssh nonexistant@pve`

This attempt should return `Permission denied (publickey)` and if no password prompt came up, everything is working.

### Firewall

The Proxmox firewall will now be set up to enable only connections from the workstation. This is all done from the Proxmox web interface.

1. Click on `Datacenter` in the `Server View` list
2. Find `Firewall` settings
3. Find Firewall `Options` below
4. Double-click on the the `Firewall` entry and change `No` to `Yes`

Firewall is disabled by default and is now enabled.

### Ubuntu cloud-init template
1. SSH back into the Proxmox server (`ssh pve`)
2. Download the Ubuntu 20.04 LTS cloud image:

```bash
wget  http://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-amd64.img
```
3. Created a new VM that will serve as a template:

```bash
qm create 9000
```
4. Import the cloud image:

```bash
qm importdisk 9000 focal-server-cloudimg-amd64.img local-lvm
```
5. You can delete the downloaded image if you wish:

```bash
rm focal-server-cloudimg-amd64.img
```

6. Configuring the VM:

```bash
qm set 9000 --name Ubuntu-20.04 --memory 2048 --net0 virtio,bridge=vmbr0 \
  --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-9000-disk-0 \
  --ide0 none,media=cdrom --ide2 local-lvm:cloudinit --boot c \
  --bootdisk scsi0 --serial0 socket --vga std --ciuser root \
  --sshkey $HOME/.ssh/authorized_keys --ipconfig0 ip=dhcp

```
### Resizing template with gParted
1. Downloading the `gparted` ISO image to resize the disk:

```bash
wget -P /var/lib/vz/template/iso \
  https://downloads.sourceforge.net/gparted/gparted-live-1.3.0-1-amd64.iso
```

2. Setting the first boot device to load `gparted`:

```bash
qm set 9000 --ide0 local:iso/gparted-live-1.3.0-1-amd64.iso,media=cdrom \
  --boot 'order=ide0;scsi0'
```

3. Resizing the disk to whatever size you choose. To add 50 GB write `+50G`, for 15 GB `+15G` and so on:

```bash
qm resize 9000 scsi0 +15G
```

In the Proxmox web interface you can now find VM 9000. All these settings can also be done via the interface. For now, start the VM and click on the `Console` in the node list.

4. Once gParted loads click `Fix` when the error prompt comes up.
5. Select `/dev/sda1` and `Resize/Move`
6. Resize the partition all the way to the right so there is no free space left.
7. `Apply` your settings and shutdown the VM.
8. Remove the gParted drive from the VM:

```bash
qm set 9000 --delete ide0
```

9. You can now conver your virtual machine into a template:

```bash
qm template 9000
```
## Installing k3s

Now that we have a VM template, we can create a cluster with one master and two worker nodes.

In the Proxmox interface we can now clone our `9000` template.

1. Right click on the template, select `Clone` and then `Linked Clone` to save disk space. This option requires the template image to be present. `Full Clones` can function independantly.
2. Enter the name of the clone to be `k3s-1`, `pve-k3s-1` or whatever you choose.
3. Start the VM and repeat for as many clones you wish. I only made 3 clones for 1 `master` and 2 `workers`
4. Wait for DHCP to assign IPs to the VMs. You can search for them through various methods. I used `Nmap` on my Solus workstation.

```bash
sudo eopkg it nmap
```

If you're using an Ubuntu or Debian based system you can install it with:

```bash
sudo apt install nmap
```

Then:

```bash
nmap -sP 192.168.1.0/24
```

*Note:* You will need to alter the IP address scheme to match yours.

Output should be something like this:

```bash
Starting Nmap 7.90 ( https://nmap.org ) at 2022-06-19 11:10 EEST
Nmap scan report for ralink.ralinktech.com (192.168.1.1)
Host is up (0.0039s latency).
Nmap scan report for 192.168.1.2
Host is up (0.035s latency).
Nmap scan report for 192.168.1.6
Host is up (0.000076s latency).
Nmap scan report for 192.168.1.7
Host is up (0.029s latency).
Nmap scan report for 192.168.1.8
Host is up (0.029s latency).
Nmap scan report for 192.168.1.9
Host is up (0.040s latency).
Nmap scan report for 192.168.1.10
Host is up (0.020s latency).
Nmap done: 256 IP addresses (7 hosts up) scanned in 3.40 seconds
```

If your doing the command right after creating and starting the clones. It should be the last three. In my case `192.168.1.8`, `192.168.1.9` and `192.168.1.10`.

5. Now it's time to edit `$HOME/.ssh/config` on the workstation and add sections for each node:

```bash
# Host SSH client config: ~/.ssh/config

Host pve-k3s-1
    Hostname 192.168.X.X
    User root
    
Host pve-k3s-2
    Hostname 192.168.X.X
    User root

Host pve-k3s-3
    Hostname 192.168.X.X
    User root
```

6. Test login to k3s-1 via ssh:

```bash
ssh pve-k3s-1
```

7. While t's time to install the k3s server on what is to be the `master` node:

```bash
curl -sfL https://get.k3s.io | sh -s - server --disable traefik
```

8. Retrieve the cluster token:

```bash
cat /var/lib/rancher/k3s/server/node-token
```

9. SSH into the `k3s-2` and `k3s-3` nodes and install the `worker` agents, replacing the value in `K3S_URL` with the node IP and `K3S_TOKEN` with the token value you copied from the previous command:

```bash
# Install K3s worker agent: fill in K3S_URL and K3S_TOKEN
curl -sfL https://get.k3s.io | K3S_URL=https://192.168.X.X:6443 K3S_TOKEN=xxxx sh
```

### Local Workstation Access

Exit the `ssh` sessions from the nodes and return to the workstation.

1. Create the `kubectl` config file:

```bash
mkdir -p $HOME/.kube && \
scp pve-k3s-1:/etc/rancher/k3s/k3s.yaml $HOME/.kube/pve-k3s && \
echo "export KUBECONFIG=$HOME/.kube/pve-k3s" >> $HOME/.bash_profile && \
export KUBECONFIG=$HOME/.kube/pve-k3s
```

2. No edit `$HOME/.kube/pve-k3s` file and replace `127.0.0.1` with the IP address of the `master` node. Also find and replace `default` with the name of the cluster `k3s-1` and save the file.

### Install `kubectl` and `helm`

#### kubectl

On my Solus workstation, I used the package from the official repository.

```bash
sudo eopkg it kubectl
```
If you're running an Ubuntu or Debian based system:

1. Update repository and install packages needed to use the Kubernetes `apt` repository:

```bash
sudo apt update
sudo apt-get install -y apt-transport-https ca-certificates curl
```
2. Download the Google Cloud public signing key:

```bash
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
```

3. Add the Kubernetes `apt` repository:

```bash
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

4. Update `apt` package index with the new repository and install `kubectl`:

```bash
sudo apt-get update
sudo apt-get install -y kubectl
```

#### helm

Consult the official [helm docs](https://helm.sh/docs/intro/install/) to choose the best way to install `helm` on your particular workstation.

I chose the binary release, as there isn't an official package in the Solus repos.

If you're on an Ubuntu or Debian based system:

```bash
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
```

### Running

1. Test that `kubectl` works.

```bash
kubectl get nodes
```

Output should be:

```bash
NAME    STATUS   ROLES                  AGE     VERSION
k3s-1   Ready    control-plane,master   5d18h   v1.23.6+k3s1
k3s-2   Ready    <none>                 5d17h   v1.23.6+k3s1
k3s-3   Ready    <none>                 5d17h   v1.23.6+k3s1
```

Don't worry of the worker nodes have `<none>` values. Everything is working.

You now have a working k3s cluster running on Proxmox.

### Create snapshots in Proxmox

In the Proxmox web interface. Click on each VM of the cluster and create snapshots:

1. Shutdown all machines and click `Take Snapshot` in the Snapshots tab. Name them something like `k3s_1_off`

2. Start all VMs and wait for them to load. Take another set of snapshots, this time with the machines running check `Include RAM` on each. Name them something like `k3s_1_on`

Make sure to create new snapshots everytime you make major changes.

## GitHub Repo
1. Created a `k3s-clustre` repo through the online interface in GitHub
2. Created a local repo in $HOME/Code/github/k3s-cluster
3. Setting up remote repo using `git`:

```bash
git init -b main
git add .
# Adds the files in the local repository and stages them for commit. To unstage a file, use 'git reset HEAD YOUR-FILE'.
git commit -m "First Commit"
# Commits the tracked changes and prepares them to be pushed to a remote repository. To remove this commit and modify the file, use 'git reset --soft HEAD~1' and commit and add the file again.
```

4. Copy remote repository URL from GitHub:

```bash
git remote add origin <REMOTE_URL>
# Sets the new remote
git remote -v
# Verifies the new remote URL
```

5. Pushing changes from local to remote:

```bash
git push origin main
```
6. Adding local file in the future:

```bash
git add .
# Adds the file to your local repository and stages it for commit. To unstage a file, use 'git reset HEAD YOUR-FILE'.
git commit -m "Commit Description"
# Commits the tracked changes and prepares them to be pushed to a remote repository. To remove this commit and modify the file, use 'git reset --soft HEAD~1' and commit and add the file again.
git push origin main
# Pushes the changes in your local repository up to the remote repository you specified as the origin
```

