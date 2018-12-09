# -*- mode: ruby -*-
# vi: set ft=ruby :

$configureBox = <<-SHELL

  # パッケージ更新
  apt-get update
  apt-get upgrade -y

  # Dockerの前提パッケージ
  apt-get install -y apt-transport-https ca-certificates curl software-properties-common
  # Dockerのレポジトリ追加
  curl -fsSL https://download.docker.com/linux/ubuntu/gpg | apt-key add -
  add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
  # Dockerのインストール
  apt-get update
  apt-get install -y docker-ce=$(apt-cache madison docker-ce | grep 17.03 | head -1 | awk '{print $3}')
  apt-mark hold docker-ce
  # vagrantユーザーをdockerグループに追加
  usermod -aG docker vagrant

  # Flannelの場合に必要
  # デフォルトが1なのでコメントアウト
  # echo net.bridge.bridge-nf-call-iptables = 1 >> /etc/sysctl.conf
  # sysctl -p

  # スワップを無効化する
  # スワップ領域がないのでコメントアウト
  # swapoff -a
  # sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab

  # Kubernetesの前提パッケージ
  # apt-get update
  # apt-get install -y apt-transport-https curl
  # Kubernetesのレポジトリ追加
  curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add -
  cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
  # kubeadm、kubelet、kubectlのインストール
  apt-get update
  # apt-get install -y kubelet=1.12.2-00 kubeadm=1.12.2-00 kubectl=1.12.2-00
  apt-get install -y kubelet kubeadm kubectl
  apt-mark hold kubelet kubeadm kubectl

  # プライベートネットワークのNICのIPアドレスを変数に格納
  IPADDR=$(ip a show enp0s8 | grep inet | grep -v inet6 | awk '{print $2}' | cut -f1 -d/)
  # kubeletがプライベートネットワークのNICにバインドするように設定
  sed -i "/KUBELET_EXTRA_ARGS=/c\KUBELET_EXTRA_ARGS=--node-ip=$IPADDR" /etc/default/kubelet
  # kubeletを再起動
  systemctl daemon-reload
  systemctl restart kubelet

SHELL

$configureMaster = <<-SHELL

  echo "This is master"

  # プライベートネットワークのNICのIPアドレスを変数に格納
  IPADDR=$(ip a show enp0s8 | grep inet | grep -v inet6 | awk '{print $2}' | cut -f1 -d/)
  # ホスト名を変数に格納
  HOSTNAME=$(hostname -s)

  # kubeadm initの実行
  # Flannelの場合
  kubeadm init --apiserver-advertise-address=$IPADDR --apiserver-cert-extra-sans=$IPADDR --node-name $HOSTNAME --pod-network-cidr=10.244.0.0/16
  # Calicoの場合
  # kubeadm init --apiserver-advertise-address=$IPADDR --apiserver-cert-extra-sans=$IPADDR --node-name $HOSTNAME --pod-network-cidr=192.168.0.0/16

  # vagrantユーザーがkubectlを実行できるようにする
  sudo --user=vagrant mkdir -p /home/vagrant/.kube
  cp -i /etc/kubernetes/admin.conf /home/vagrant/.kube/config
  chown $(id -u vagrant):$(id -g vagrant) /home/vagrant/.kube/config

  # Flannelのインストール
  export KUBECONFIG=/etc/kubernetes/admin.conf
  kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/bc79dd1505b0c8681ece4de4c0d86c5cd2643275/Documentation/kube-flannel.yml
  # Calicoのインストール
  # export KUBECONFIG=/etc/kubernetes/admin.conf
  # kubectl apply -f https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/hosted/rbac-kdd.yaml
  # kubectl apply -f https://docs.projectcalico.org/v3.3/getting-started/kubernetes/installation/hosted/kubernetes-datastore/calico-networking/1.7/calico.yaml

  # kubectl joinコマンドを保存する
  kubeadm token create --print-join-command > /etc/kubeadm_join_cmd.sh
  chmod +x /etc/kubeadm_join_cmd.sh

  # sshでのパスワード認証を許可する
  sed -i "/^[^#]*PasswordAuthentication[[:space:]]no/c\PasswordAuthentication yes" /etc/ssh/sshd_config
  systemctl restart sshd

SHELL

$configureNode = <<-SHELL

  echo "This is worker"

  apt-get install -y sshpass
  sshpass -p "vagrant" scp -o StrictHostKeyChecking=no vagrant@192.168.33.11:/etc/kubeadm_join_cmd.sh .
  # sshpass -p "vagrant" scp -o StrictHostKeyChecking=no vagrant@172.16.33.11:/etc/kubeadm_join_cmd.sh .
  sh ./kubeadm_join_cmd.sh

SHELL

Vagrant.configure(2) do |config|

  (1..2).each do |i|

    if i == 1 then
      vm_name = "master"
    else
      vm_name = "node#{i-1}"
    end
      
    config.vm.define vm_name do |s|

      # ホスト名
      s.vm.hostname = vm_name
      # ノードのベースOSを指定
      s.vm.box = "ubuntu/xenial64"
      # ネットワークを指定
      # pod-network-cidrと重ならないように注意
      private_ip = "192.168.33.#{i+10}"
      # private_ip = "172.16.33.#{i+10}"
      s.vm.network "private_network", ip: private_ip

      # ノードのスペックを指定
      s.vm.provider "virtualbox" do |v|
        v.gui = false        
        if i == 1 then
          v.cpus = 2
          v.memory = 1024
        else
          v.cpus = 1
          v.memory = 1024
        end
      end

      # 共通のプロビジョニング
      s.vm.provision "shell", inline: $configureBox

      if i == 1 then
        # Masterのプロビジョニング
        s.vm.provision "shell", inline: $configureMaster
      else
        # Nodeのプロビジョニング
        s.vm.provision "shell", inline: $configureNode
      end
    
    end
  end
end