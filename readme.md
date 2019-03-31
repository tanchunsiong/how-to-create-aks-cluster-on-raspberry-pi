#download image and burn using jessie'

https://downloads.raspberrypi.org/raspbian_lite/images/raspbian_lite-2017-07-05/

ENABLE SSH, change PI name
sudo raspi-config

This is how I solved this problem. Took a while to figure out so hopefully this post helps somebody.

Connect your Raspberry Pi 3+ to your Router with an Ethernet cable. Run sudo apt-get update and then sudo apt-get upgrade. This should just work out of the box.
Run sudo iwlist wlan0 scanning | grep ESSID. If your WiFi name is there then move on to next step.
Run wpa_passphrase "mywireless_ssid" "yourpassphrase" | sudo tee -a /etc/wpa_supplicant/wpa_supplicant.conf. This will add create something like this in your wpa_supplicant.conf file:

network={
    ssid="SSID"
    #psk="PASSPHRASE"
    psk=38497220976092fc2707a838e4d4385019256149f99f935be22c90159d3b8373
}
Delete the #psk="PASSPHRASE" line and save the file.

Then make sure your /etc/network/interfaces file looks like this:

auto lo

iface lo inet loopback
iface eth0 inet dhcp

allow-hotplug wlan0
auto wlan0


iface wlan0 inet dhcp
wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf

#Then reboot the pi 
sudo reboot

Then hopefully you can run ifconfig and your wlan0 will have ip address.

A lot of this I got from this post.

#Install Docker
curl -sSL get.docker.com | sh &&  sudo usermod pi -aG docker
newgrp docker

#Disable Swap. Important, you'll get errors in Kuberenetes otherwise
sudo dphys-swapfile swapoff &&  sudo dphys-swapfile uninstall &&  sudo update-rc.d dphys-swapfile remove
#Go edit /boot/cmdline.txt with your favorite editor, or use
sudo nano /boot/cmdline.txt
#and add this at the very end. Don't press enter.
cgroup_enable=cpuset cgroup_enable=memory
#Install Kubernetes
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add - &&  echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list &&  sudo apt-get update -q &&  sudo apt-get install -qy kubeadm 


sudo reboot



#MASTER/BOSS NODE
#After ssh'ing into my main node, I used /ifconfig eth0 to figure out what the IP adresss was. Ideally you want this to be static (not changing) or at least a #static lease. I logged into my router and set it as a static lease, so my main node ended up being 192.168.170.2, and .1 is the router itself.

#Then I initialized this main node
#
sudo kubeadm config images pull -v3

sudo sed -i 's/failureThreshold: 8/failureThreshold: 20/g' /etc/kubernetes/manifests/kube-apiserver.yaml && \
sudo sed -i 's/initialDelaySeconds: [0-9]\+/initialDelaySeconds: 360/' /etc/kubernetes/manifests/kube-apiserver.yaml

sudo kubeadm init --apiserver-advertise-address=192.168.31.154 --token-ttl=0  --pod-network-cidr=10.244.0.0/16
#sudo kubeadm reset to undo
#This took a WHILE. Like 10-15 min, so be patient.

#Kubernetes uses this admin.conf for a ton of stuff, so you're going to want a copy in your $HOME folder so you can call "kubectl" easily later, copy it and take #ownership.

mkdir -p $HOME/.kube 
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config 
sudo chown $(id -u):$(id -g) $HOME/.kube/config


kubectl apply -f https://git.io/weave-kube-1.6

#When this is done, you'll get a nice print out with a ton of info and a token you have to save. Save it all. I took a screenshot.

#for workernodes only

kubeadm join 192.168.31.154:6443 --token u6h0n6.t5gz3w502t22hgmz \
    --discovery-token-ca-cert-hash sha256:e24645acf925f1ecb5f58277c781e31a1ebd99d97127ecce0c8b515f9b906df7



#create clusterole binding
#create dashboard-admin.yaml (development release)
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard-head
  labels:
    k8s-app: kubernetes-dashboard-head
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard-head
  namespace: kube-system

#run
kubectl create -f dashboard-admin.yaml

#create dashboard-admin-user.yaml (development release)
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kubernetes-dashboard-head
  namespace: kube-system


#This is the development/alternative dashboard which has TLS disabled and is easier to use.

kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/aio/deploy/alternative/kubernetes-dashboard-arm-head.yaml

kubectl config set-credentials kubernetes-dashboard-head --token=eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC1oZWFkLXRva2VuLWpkc3hiIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6Imt1YmVybmV0ZXMtZGFzaGJvYXJkLWhlYWQiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiIxNWFhN2VmMC01M2I1LTExZTktODRmZS1iODI3ZWJhMjA1YzciLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06a3ViZXJuZXRlcy1kYXNoYm9hcmQtaGVhZCJ9.s3wro5IWXgMM5xGF2J9fJ-k1Dq5My6Cw7EbPn6ESJyR5Hxs4-gD0cwp6VUPKnSfswPYIzJ086KKFKNHicw3rwZNR7alRG4y7oe-tiumSEVtpFT11qKbG2jxSKMrdiWK4vElF_K1FId-W1VUWvJ77wBx2o155m735zsFMuomC960ScHrT0xA0ki_qkQeeeahlRDVtcuixqtKK7jr3GagltHAZ-R8w9Aa5_p6vjd5DmwyWqCv6U8wZH6_ybnflZo_EaSPqLvAjBu-ZCXHzkdHzSyefDlTILvRpDUpBQpHx1xblDAbL5oPM807o3PI0FnfPYqZMTWsL40aK0Y6JcMH2Gg


#install kubectl on your windows machine
https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-with-chocolatey-on-windows

#make sure you already have /user/.kube/config 
#name must be config
#you can troubleshoot by typing
kubectl config view

#make sure dashboard is not in slave nodes
kubectl get pods -o wide --all-namespaces
kubectl get serviceaccounts -n kube-system
kubectl get role,rolebinding -n kube-system |grep kubernetes-dashboard-minimal-head

login at http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard-head:/proxy/#/overview

kubectl -n kube-system get secret
kubectl -n kube-system describe secret kubernetes-dashboard-head-token-jdsxb

how to sign in via cluster admin?
kubectl config set-credentials cluster-admin --token=eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJrdWJlcm5ldGVzLWRhc2hib2FyZC1oZWFkLXRva2VuLWpkc3hiIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6Imt1YmVybmV0ZXMtZGFzaGJvYXJkLWhlYWQiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiIxNWFhN2VmMC01M2I1LTExZTktODRmZS1iODI3ZWJhMjA1YzciLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06a3ViZXJuZXRlcy1kYXNoYm9hcmQtaGVhZCJ9.s3wro5IWXgMM5xGF2J9fJ-k1Dq5My6Cw7EbPn6ESJyR5Hxs4-gD0cwp6VUPKnSfswPYIzJ086KKFKNHicw3rwZNR7alRG4y7oe-tiumSEVtpFT11qKbG2jxSKMrdiWK4vElF_K1FId-W1VUWvJ77wBx2o155m735zsFMuomC960ScHrT0xA0ki_qkQeeeahlRDVtcuixqtKK7jr3GagltHAZ-R8w9Aa5_p6vjd5DmwyWqCv6U8wZH6_ybnflZo_EaSPqLvAjBu-ZCXHzkdHzSyefDlTILvRpDUpBQpHx1xblDAbL5oPM807o3PI0FnfPYqZMTWsL40aK0Y6JcMH2Gg

https://www.hanselman.com/blog/HowToBuildAKubernetesClusterWithARMRaspberryPiThenRunNETCoreOnOpenFaas.aspx