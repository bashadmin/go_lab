# go_lab

## add git branch to bash profile
add to your .bashrc file

```
parse_git_branch() {
  git branch 2> /dev/null | sed -e '/^[^*]/d' -e 's/* \(.*\)/ (\1)/'
}

export PS1="\u@\h \W\[\033[32m\]\$(parse_git_branch)\[\033[00m\] $ "
```
run this command after you finish the edit
```
source ~/.bashrc
```


## Setting up okd

After the installation has completed, login, and update the OS.

```
sudo subscription-manager repos --enable codeready-builder-for-rhel-8-$(arch)-rpms
sudo dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
sudo dnf install epel-release -y
sudo dnf update -y
```
Setup XRDP for Remote Access from Home Network
```
sudo dnf install xrdp tigervnc-server -y
sudo systemctl enable --now xrdp
sudo firewall-cmd --zone=public --permanent --add-port=3389/tcp
sudo firewall-cmd --reload
```
Download Google Chrome rpm and install along with git

Open a terminal on the okd4-services VM and clone the okd4_files repo that contains the DNS, HAProxy, and install-conf.yaml example files:
```
cd
git clone https://github.com/cragr/okd4_files.git
cd okd4_files
```
Install bind (DNS)
```
sudo dnf install bind bind-utils -y
```
Copy the named config files and zones:
```
sudo cp named.conf /etc/named.conf
sudo mkdir -p /etc/named/zones
sudo cp named.conf.local /etc/named/
sudo cp db* /etc/named/zones
```
Create firewall rules:
```
sudo firewall-cmd --permanent --add-port=53/udp
sudo firewall-cmd --reload
```
Change the DNS on the okd4-service NIC that is attached to the VM Network (not OKD) to 127.0.0.1.

Restart the network services on the okd4-services VM:
```
sudo systemctl restart NetworkManager
```
Test DNS on the okd4-services.
```
dig okd.local
dig -x 192.168.100.210
```

Install HAProxy:
```
sudo dnf install haproxy -y
```
Copy haproxy config from the git okd4_files directory :
```
sudo cp haproxy.cfg /etc/haproxy/haproxy.cfg
```
Start, enable, and verify HA Proxy service:
```
sudo setsebool -P haproxy_connect_any 1
sudo systemctl enable haproxy
sudo systemctl start haproxy
sudo systemctl status haproxy
```
Add OKD firewall ports:
```
sudo firewall-cmd --permanent --add-port=6443/tcp
sudo firewall-cmd --permanent --add-port=22623/tcp
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```
Install Apache/HTTPD
```
sudo dnf install -y httpd
```
Change httpd to listen port to 8080:
```
sudo sed -i 's/Listen 80/Listen 8080/' /etc/httpd/conf/httpd.conf
```
Enable and Start httpd service/Allow port 8080 on the firewall:
```
sudo setsebool -P httpd_read_user_content 1
sudo systemctl enable httpd
sudo systemctl start httpd
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload
```
Test the webserver:
```
curl localhost:8080
```
Download the 4.x version of the oc client and openshift-install from the OKD [releases page](https://github.com/openshift/okd/releases). 

*learned how to make words hyperlinked from [-- this website --](https://anvilproject.org/guides/content/creating-links)*

Example:
```
cd
wget https://github.com/openshift/okd/releases/download/4.5.0-0.okd-2020-07-29-070316/openshift-client-linux-4.5.0-0.okd-2020-07-29-070316.tar.gz
wget https://github.com/openshift/okd/releases/download/4.5.0-0.okd-2020-07-29-070316/openshift-install-linux-4.5.0-0.okd-2020-07-29-070316.tar.gz
```
Current version:
```
cd
wget https://github.com/okd-project/okd/releases/download/4.11.0-0.okd-2022-12-02-145640/openshift-client-linux-4.11.0-0.okd-2022-12-02-145640.tar.gz
wget https://github.com/okd-project/okd/releases/download/4.11.0-0.okd-2022-12-02-145640/openshift-install-linux-4.11.0-0.okd-2022-12-02-145640.tar.gz
```

Extract the okd version of the oc client and openshift-install:
```
tar -zxvf openshift-client-linux-4.11.0-0.okd-2022-12-02-145640.tar.gz
tar -zxvf openshift-install-linux-4.11.0-0.okd-2022-12-02-145640.tar.gz
```
Move the kubectl, oc, and openshift-install to /usr/local/bin and show the version:
```
sudo mv kubectl oc openshift-install /usr/local/bin/
oc version
openshift-install version
```


https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/37.20221106.3.0/x86_64/fedora-coreos-37.20221106.3.0-metal.x86_64.raw.xz


The latest and recent releases are available at https://quay.io/openshift-release-dev/ocp-release

https://amd64.origin.releases.ci.openshift.org/

https://github.com/okd-project/okd/releases

Generate an SSH key if you do not already have one.
```
ssh-keygen
```

Create an install directory and copy the install-config.yaml file:
```
cd
mkdir install_dir
cp okd4_files/install-config.yaml ./install_dir
```
Edit the install-config.yaml in the install_dir, insert your pull secret and ssh key, and backup the install-config.yaml as it will be deleted in the next step:
```
vim ./install_dir/install-config.yaml
cp ./install_dir/install-config.yaml ./install_dir/install-config.yaml.bak
```
Generate the Kubernetes manifests for the cluster, ignore the warning:
```
openshift-install create manifests --dir=install_dir/
```
Modify the cluster-scheduler-02-config.yaml manifest file to prevent Pods from being scheduled on the control plane machines:
```
sed -i 's/mastersSchedulable: true/mastersSchedulable: false/' install_dir/manifests/cluster-scheduler-02-config.yml
```

```
sudo cp -R install_dir/* /var/www/html/okd4/
sudo chown -R apache: /var/www/html/
sudo chmod -R 755 /var/www/html/
```

Now you can create the ignition-configs:
```
openshift-install create ignition-configs --dir=install_dir/
```
Note: If you reuse the install_dir, make sure it is empty. Hidden files are created after generating the configs, and they should be removed before you use the same folder on a 2nd attempt.

Create okd4 directory in /var/www/html:
```
sudo mkdir /var/www/html/okd4
```
Copy the install_dir contents to /var/www/html/okd4 and set permissions:
```
sudo cp -R install_dir/* /var/www/html/okd4/
sudo chown -R apache: /var/www/html/
sudo chmod -R 755 /var/www/html/
```
Test the webserver:
```
curl localhost:8080/okd4/metadata.json
```
Download the [Fedora CoreOS](https://getfedora.org/coreos/download/) bare-metal bios image and sig files and shorten the file names:
```
cd /var/www/html/okd4/
sudo wget https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/37.20221106.3.0/x86_64/fedora-coreos-37.20221106.3.0-metal.x86_64.raw.xz
sudo wget https://builds.coreos.fedoraproject.org/prod/streams/stable/builds/37.20221106.3.0/x86_64/fedora-coreos-37.20221106.3.0-metal.x86_64.raw.xz.sig
sudo mv fedora-coreos-37.20221106.3.0-metal.x86_64.raw.xz fcos.raw.xz
sudo mv fedora-coreos-37.20221106.3.0-metal.x86_64.raw.xz.sig fcos.raw.xz.sig
sudo chown -R apache: /var/www/html/
sudo chmod -R 755 /var/www/html/
```

## Starting the bootstrap node:
Power on the odk4-bootstrap VM. Press the TAB key to edit the kernel boot options and add the following:
```
coreos.inst.install_dev=/dev/sda coreos.inst.image_url=http://192.168.1.210:8080/okd4/fcos.raw.xz coreos.inst.ignition_url=http://192.168.1.210:8080/okd4/bootstrap.ign
```

## Starting the control plane nodes:
Power on the control-plane nodes and press the TAB key to edit the kernel boot options and add the following, then press enter:
```
coreos.inst.install_dev=/dev/sda coreos.inst.image_url=http://192.168.1.210:8080/okd4/fcos.raw.xz coreos.inst.ignition_url=http://192.168.1.210:8080/okd4/master.ign
```

## Starting the compute nodes:
Power on the control-plane nodes and press the TAB key to edit the kernel boot options and add the following, then press enter:
```
coreos.inst.install_dev=/dev/sda coreos.inst.image_url=http://192.168.1.210:8080/okd4/fcos.raw.xz coreos.inst.ignition_url=http://192.168.1.210:8080/okd4/worker.ign
```

## Monitor the bootstrap installation:
You can monitor the bootstrap process from the okd4-services node:
```
openshift-install --dir=install_dir/ wait-for bootstrap-complete --log-level=info
```
or
```
openshift-install --dir=install_dir/ wait-for bootstrap-complete --log-level=debug
```

Once the bootstrap process is complete, which can take upwards of 30 minutes, you can shutdown your bootstrap node. Now is a good time to edit the /etc/haproxy/haproxy.cfg, comment out the bootstrap node, and reload the haproxy service.
```
sudo sed '/ okd4-bootstrap /s/^/#/' /etc/haproxy/haproxy.cfg
sudo systemctl reload haproxy
```