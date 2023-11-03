DevOps Challenge / İlhan Furkan Sadıkoğlu Kubernetes ve Helm kurulumu ![](Aspose.Words.9a88fd77-cf8e-42af-91b3-78a48d1881df.001.png)

1  sudo su![](Aspose.Words.9a88fd77-cf8e-42af-91b3-78a48d1881df.002.png)
1  touch k8s\_install.sh
1  chmod u+x k8s\_install.sh
1  nano k8s\_install.sh

HOSTNAME değişkeninin düzeltilmesi gerekir. **k8s\_install.sh:**

1  #! /bin/bash -xe![](Aspose.Words.9a88fd77-cf8e-42af-91b3-78a48d1881df.003.png)
1  apt-get update -y
1  apt-get upgrade -y
1  apt-get -y install linux-image-generic-hwe-18.04
1  apt-get install -y sshpass
1  ssh-keygen -b 2048 -t rsa -f ~/.ssh/id\_rsa -q -N ""
1  sed -ie 's/PasswordAuthentication no/PasswordAuthentication yes/g' /etc/ssh/sshd\_config
1  sed -ie 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/g' /etc/ssh/sshd\_config
1  systemctl restart sshd.service
1  echo wordpress > /root/password.txt
1  echo "root:wordpress" | chpasswd
1  su -c "sshpass -f /root/password.txt ssh-copy-id root@HOSTNAME -o StrictHostKeyChecking=no"
1  apt -y install python3-pip
1  cd /root/
1  git clone https://github.com/kubernetes-sigs/kubespray.git
1  cd /root/kubespray
1  pip3 install -r requirements.txt
1  cp -rfp inventory/sample inventory/mycluster
1  PRVT=$(hostname -I)
1  declare -a IPS=($PRVT)
1  CONFIG\_FILE=inventory/mycluster/hosts.yaml python3 contrib/inventory\_builder/inventory.py $IPS
1  sed -ie 's/node1/HOSTNAME/g' inventory/mycluster/hosts.yaml
1  #sed -ie 's/PasswordAuthentication yes/PasswordAuthentication no/g' /etc/ssh/sshd\_config
1  #sed -ie 's/PermitRootLogin yes/#PermitRootLogin prohibit-password/g' /etc/ssh/sshd\_config
1  systemctl restart sshd.service
1  ansible-playbook -i inventory/mycluster/hosts.yaml --become-user=root cluster.yml
1  curl -fsSL -o get\_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
1  chmod 700 get\_helm.sh
1  ./get\_helm.sh

1 bash k8s\_install.sh

Not: Version check’te hata verirse requirements.txt ve cluster.yml’daki version değerlerini düzeltmek gerekir. (ansible 6.7.0, core: 2.13.7)

Not: kubespray altındaki playbooks altında ansible\_version.yml altında minimal ve maximal değerin arasında kalması gereken değer: 2.13.7 dir. yani 2.13.0 ve 2.14.0 verilebilir. Düzeltmeler sonrasında işlemlere 16. satırdaki  cd /root/kubespray komutundan devam edilir.

K9S kurulumu (Management UI) ![](Aspose.Words.9a88fd77-cf8e-42af-91b3-78a48d1881df.004.png)

1  sudo su![](Aspose.Words.9a88fd77-cf8e-42af-91b3-78a48d1881df.005.png)
1  curl -sS https://webi.sh/k9s | sh
1  source ~/.config/envman/PATH.env

Postgresql Installation ![](Aspose.Words.9a88fd77-cf8e-42af-91b3-78a48d1881df.006.png)

1  helm repo add bitnami https://charts.bitnami.com/bitnami![](Aspose.Words.9a88fd77-cf8e-42af-91b3-78a48d1881df.007.png)
1  nano postgres-pv.yaml

**postgres-pv.yaml:**

1  apiVersion: v1![](Aspose.Words.9a88fd77-cf8e-42af-91b3-78a48d1881df.008.png)
1  kind: PersistentVolume
1  metadata:
1  name: postgresql-pv
1  labels:
1  type: local
1  spec:
1  storageClassName: manual
1  capacity:
10  storage: 10Gi
10  accessModes:
10  - ReadWriteOnce
10  hostPath:
10  path: "/mnt/data1"
1  kubectl apply -f postgres-pv.yaml![](Aspose.Words.9a88fd77-cf8e-42af-91b3-78a48d1881df.009.png)
1  nano postgres-pvc.yaml

**postgres-pvc.yaml:**

1  apiVersion: v1![](Aspose.Words.9a88fd77-cf8e-42af-91b3-78a48d1881df.010.png)
1  kind: PersistentVolumeClaim
1  metadata:
1  name: postgresql-pv-claim
1  spec:
1  storageClassName: manual
1  accessModes:
1  - ReadWriteOnce
1  resources:
10  requests:
10  storage: 10Gi
1  kubectl apply -f postgres-pvc.yaml![](Aspose.Words.9a88fd77-cf8e-42af-91b3-78a48d1881df.011.png)
1  kubectl get pv
1  kubectl get pvc

Son 2 komutun çıktısında volume'ün bound olduğundan emin olunmalıdır.

Ardından:

1 helm install postgresql bitnami/postgresql --set persistence.existingClaim=postgresql-pv-claim --set volumePermissions.enabled=true![](Aspose.Words.9a88fd77-cf8e-42af-91b3-78a48d1881df.012.png)

Bu adımların ardından kurulum tamamlanır. K9S üzerinden kontrol yapılabilir.

Root parolasını almak için:

1 export POSTGRES\_PASSWORD=$(kubectl get secret --namespace default psql-test-postgresql -o jsonpath="{.data.postgresql-password}" | base64 --decode)![](Aspose.Words.9a88fd77-cf8e-42af-91b3-78a48d1881df.013.png)

Son olarak **k edit svc postgresql** komutu ile service düzenlenir ve nodeport ayarlanır:

1  nodePort=31532![](Aspose.Words.9a88fd77-cf8e-42af-91b3-78a48d1881df.014.png)
1  type: NodePort

Postgre’ye şu komutla bağlanılır:

1 psql --host 127.0.0.1 -U postgres -d postgres -p 31532![](Aspose.Words.9a88fd77-cf8e-42af-91b3-78a48d1881df.015.png)

**Backup**

Aşağıdaki işlemler backup sunucusunda yapılır.  USERNAME ,  KUBERNETESIP ,  /path/to/db-backups ve  /path/to/mnt düzenlenmelidir. Ayrıca backup sunucusunun public key’i kubernetes makinesine eklenmelidir. Backup sunucusunda pg\_dump’un aynı versiyonu bulunmalıdır.

1  mkdir /path/to/db-backups![](Aspose.Words.9a88fd77-cf8e-42af-91b3-78a48d1881df.016.png)
1  touch db-backups.sh
1  chmod u+x db-backups.sh
1  nano db-backups.sh
1  #! /bin/bash -xe![](Aspose.Words.9a88fd77-cf8e-42af-91b3-78a48d1881df.017.png)
1  ssh USERNAME@KUBERNETESIP 'tar -zcf postgres\_data-$(hostname)-$(date +%d-%m-%y).gz /path/to/mnt && echo "data alındı"'
1  PGPASSWORD="$POSTGRES\_PASSWORD" pg\_dumpall -h KUBERNETESIP -U postgres -p 31532 -f postgres\_dump-$(hostname)-$(date +%d-%m-%y).sql'

1 crontab -e

Aşağıdaki satır eklenir:

1 0 0 \* \* \* bash /path/to/db-backups/db-backups.sh![](Aspose.Words.9a88fd77-cf8e-42af-91b3-78a48d1881df.018.png)

Redis Kurulumu ![](Aspose.Words.9a88fd77-cf8e-42af-91b3-78a48d1881df.019.png)

1  helm install redis bitnami/redis![](Aspose.Words.9a88fd77-cf8e-42af-91b3-78a48d1881df.020.png)
1  export REDIS\_PASSWORD=$(kubectl get secret --namespace default redis -o jsonpath="{.data.redis-password}" | base64 --decode)

1 nano redis-pv.yaml

**redis-pv.yaml:**

1  apiVersion: v1![](Aspose.Words.9a88fd77-cf8e-42af-91b3-78a48d1881df.021.png)
1  kind: PersistentVolume
1  metadata:
1  name: redis-data-redis-master0
1  spec:
1  capacity:
1  storage: 8Gi
1  accessModes:
1  - ReadWriteOnce
1  hostPath:
1  path: "/storage/data-master0" 12
13  ---![](Aspose.Words.9a88fd77-cf8e-42af-91b3-78a48d1881df.022.png)
13  apiVersion: v1
13  kind: PersistentVolume
13  metadata:
13  name: redis-data-redis-replicas0
13  spec:
13  capacity:
13  storage: 8Gi
13  accessModes:
13  - ReadWriteOnce
13  hostPath:
13  path: "/storage/data-replicas0"

1 kubectl apply -f redis-pv.yaml

K9s üzerinden podun çalıştığı teyit edilir.

Son olarak **k edit svc redis** komutu ile service düzenlenir ve nodeport ayarlanır:

1  nodePort=31679
1  type: NodePort

Test için redis-cli kullanılabilir.

1  sudo apt install redis-tools
1  redis-cli -h KUBERNETESIP -p 31679 ping
