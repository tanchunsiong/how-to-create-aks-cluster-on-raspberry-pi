 # Drain and delete the nodes (for each node you have)
    kubectl drain kubenode1 --delete-local-data --force --ignore-daemonsets
    kubectl delete node kubenode1

    # Reset the deployment
    sudo kubeadm reset

    # On each node

    ## Reset the nodes and weave
    sudo curl -L git.io/weave -o /usr/local/bin/weave
    sudo chmod a+x /usr/local/bin/weave
    sudo kubeadm reset
    sudo weave reset --force

    ## Clean weave binaries
    sudo rm /opt/cni/bin/weave-*

    ## Flush iptables rules on all nodes and restart Docker
    iptables -P INPUT ACCEPT
    iptables -P FORWARD ACCEPT
    iptables -P OUTPUT ACCEPT
    iptables -t nat -F
    iptables -t mangle -F
    iptables -F
    iptables -X
    systemctl restart docker