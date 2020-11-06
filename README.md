# ubuntu-pies
# Goal
Automate install and configure Raspberry Pi with Ubuntu 20.04 LTS 64-bit OS as a Kubernetes cluster.
At current time, this means only Raspberry Pi 3 and 4 are compatible with the 64-bit OS.
This will also perform the installs in a headless configuration.
# Steps
The steps were run on a Mac using Ubuntu imager (to build the prepare the SD card), ssh, and ansible.
Hopefully these tasks remain OS neutral, but YMMV.
1. Per the [Raspberry Pi Ubuntu installation instructions](https://ubuntu.com/tutorials/how-to-install-ubuntu-on-your-raspberry-pi#1-overview),
prepare the SD card with [Step #2 Prepare the SD Card](https://ubuntu.com/tutorials/how-to-install-ubuntu-on-your-raspberry-pi#2-prepare-the-sd-card)
1. If your raspberry pi will only be connected via wifi, keep the SD card mounted to your host.
If the you or the Ubuntu imager has already unmounted it, remount it by reinserting the SD card back into your host.
   ```bash
   cd /Volumes/system-boot
   # edit the network-config file and add your wifi configuration (plenty of examples on the web)
   
   # this should take you out of the previous directory and to your home directory
   cd
   
   # on a mac, to unmount the SD card do something like this (YMMV for the disk2):
   diskutil unmountDisk /dev/disk2
   ```
1. Insert the SD card into your Raspberry Pi, provide network to your pi and then power.
It should take quite a while, but you pi should boot, join the network, and then install updates.
1. Find your pi's IP address on the network.
This is problematic.
In the past, I would have recommended using **arp**, but you may need to install a tool like **nmap** and run something like:
   ```bash
   nmap -sV -p 22 10.0.4.0/24 -open
   ```
   and try to locate your pi.
1. install ansible galaxy community on install host
   ```bash
   ansible-galaxy collection install community.general
   ```
1. Connect remotely with your pi:
   ```
    ssh ubuntu@YOUR_PI_IP_ADDRESS_FROM_PREVIOUS_STEP
   ```
   Initially, the password for the ubuntu user will be *ubuntu*, and you will be asked to change it on first entry.
1. Allow the pi to finish its unattended updates.
Using a tool like *htop*, *top*, or a form of the *ps* command, monitor the progress of the unattended updates, and let finish.
Once it seems the updates have finished, logout, log back in, and the login banner should advise you of the need to reboot the pi to finish installing the updates:
   ```bash
   # to reboot your pi
   sudo reboot
   ```
   It will probably take a minute or so for the pi to reboot and then *ssh* back into your pi:
   ```bash
   ssh ubuntu@YOUR_PI_IP_ADDRESS_FROM_PREVIOUS_STEP
   ```
1. Set the hostname for your pi:
   ```bash
   sudo hostnamectl set-hostname NEW_PI
   ```
1. Reboot may be necessary
   ```
   sudo reboot
   ```
1. Make your pi IP address static. Easiest way is to set IP address in your internet router. YMMV.
1. Enable ssh access without password for install host (from install host -- not new pi):
   ```bash
   ssh-copy-id ubuntu@NEW_PI
   ```

# Notes

## Turn on some key addons
```bash
microk8s dns storage ingress
# recommend restart after installing any addons: microk8s stop;microk8s start
```
*Recommend microk8s restart after installing any addons*
```bash
microk8s stop;microk8s start
```

## Turn on dashboard
```bash
microk8s enable dashboard
```
*Recommend microk8s restart after installing any addons*
```bash
microk8s stop;microk8s start
```
#### By the book
Supposedly the following executed on controller allow you to get a token you can use to log into the dashboard:
```bash
token=$(microk8s kubectl -n kube-system get secret | grep default-token | cut -d " " -f1)
microk8s kubectl -n kube-system describe secret $token
```
It generates a token but when I supplied the token on the dashboard, 
it responded with no errors BUT did not redirect to dashboard.
#### Workaround
So since the cluster runs on local network AND I will start/stop dashboard only when needed,
I bypassed authentication on dashboard by running this on the controller:
```bash
kubectl edit deployment/kubernetes-dashboard --namespace=kube-system
```
This puts you into edit mode. Make the folling edits:
```yaml
    spec:
      containers:
      - args:
        - --enable-skip-login
        - --auto-generate-certificates
        - --namespace=kube-system
        image: kubernetesui/dashboard:v2.0.0
```

#### Start dashboard with proxy to access the dashboard from outside localhost
```bash
microk8s.kubectl proxy --accept-hosts=.* --address=0.0.0.0 &
```
#### Your browser *should* be able to reach dashboard now:
```http request
http://UBUNTU_IP:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/
```

## Add cluster node
```bash
microk8s add-node
```
Output will give command such as: 
```bash
microk8s join 10.0.4.109:25000/15cc8de89ef2b8687dcce996ed8163c2
```
*Run on the 'to be joined' node*.
Once joined, view all nodes:
```bash
microk8s.kubectl get nodes
```

## Alias microk8s.kubectl with kubectl
```bash
sudo snap alias microk8s.kubectl kubectl
```
This will allow you to use just *kubectl* going forward rather than *microk8s.kubectl*.

## Is v1beta1.metrics.k8s.io throwing errors on controller or workers?
```bash
kubectl get apiservices
```
If output shows FailedDiscoveryCheck for metric server like:
```bash
v1beta1.metrics.k8s.io      kube-system/metrics-server      False (FailedDiscoveryCheck)
```
*You may need to restart microk8s restart after installing any addons.*
```bash
microk8s stop;microk8s start
```
*If rash persists, you may need to review NO_PROXY setting in /var/snap/microk8s/current/args/containerd-env.*

## Add clustered nginx
#### Create a deployment (will spin up single pod):
```bash
kubectl create deployment --image nginx pi-nginx
```
#### Verify pod is running:
```bash
kubectl get pods
```
#### Verify deployment was created:
```bash
kubectl get deployment
```
#### Fixed scale
Scale the deployment to ensure there are multiple nginx pods (example starts 4):
```bash
kubectl scale deployment --replicas 4 pi-nginx
```
#### or autoscale
Example values used:
```
kubectl autoscale deployment.v1.apps/pi-nginx  --min=2 --max=20 --cpu-percent=80
```
#### Verify additional pods are running:
```bash
kubectl get pods
```
#### expose the nginx cluster
##### If using compatible cloud load balancer
```bash
kubectl expose deployment pi-nginx --port=80 --type=LoadBalancer
```
##### or expose the port locally
```bash
kubectl expose deployment pi-nginx --type=NodePort --port=80
```
#### to verify service:
```bash
kubectl describe services pi-nginx
```
#### to reveal port:
```bash
kubectl describe services pi-nginx | grep ^NodePort
```
#### to adjust scaling you may have to delete deployment and recreate
```bash
kubectl delete deployment pi-nginx
kubectl create deployment --image nginx pi-nginx
```
Will then have to re-expose deployment (above).

## Start interactive shell on cluster
For debugging purposes (especially when it comes to DNS issues), you can start an interactive container:
```
kubectl run my-shell --rm -i --tty --image ubuntu -- bash
```
When you exit the shell, the container should clean up after itself.

