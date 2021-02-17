# Snacka Kubernetes Security 40min


https://www.cisecurity.org

The Level 1 profile is considered a base recommendation that can be implemented fairly promptly and is designed to not have an extensive performance impact. The intent of the Level 1 profile benchmark is to lower the attack surface of your organization while keeping machines usable and not hindering business functionality.

The Level 2 profile is considered to be "defense in depth" and is intended for environments where security is paramount. The recommendations associated with the Level 2 profile can have an adverse effect on your organization if not implemented appropriately or without due care.

# Prereq to run demo:
<PRE>
sudo snap install helm --classic
sudo apt-get install unzip
sudo apt-get install -y openjdk-11-jdk
export JAVA_PATH=/usr/lib/jvm/java-11-openjdk-amd64/bin/
</PRE>

# CIS Ubuntu
Copy link v4 from Mail
<PRE>
wget https://learn.cisecurity.org/e/799323/l-799323-2019-11-15-3v7x/2mnnf/126489250?h=_X4qaniCm1_pYYKOtD5RBES62ujGVaFzfeq-ssDWNSc

mv '126489250?h=_X4qaniCm1_pYYKOtD5RBES62ujGVaFzfeq-ssDWNSc' cis.zip
unzip cis.zip
sudo bash Assessor-CLI.sh -i
</PRE>

# Trivy
<PRE>
sudo apt-get install -y wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy

trivy centos:7
</PRE>
# Kube-bench
<PRE>
curl -L https://github.com/aquasecurity/kube-bench/releases/download/v0.3.1/kube-bench_0.3.1_linux_amd64.deb -o kube-bench_0.3.1_linux_amd64.deb
sudo apt install ./kube-bench_0.3.1_linux_amd64.deb -f

kube-bench master
</PRE>

# Falco
<PRE>
sudo snap install helm --classic
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update
helm install falco falcosecurity/falco
helm ls
kubectl get ds
kubectl logs -f falco-gst58 # Replace with one of your pods
</PRE>


### Open another terminal window:
<PRE>
kubectl run andy --image=centos:7 -- sleep 100000
kubectl get pods 
kubectl exec -it andy -- bash
</PRE>

# Tracee (https://github.com/aquasecurity/tracee)
<PRE>
docker run --name tracee --rm --privileged --pid=host -v /lib/modules/:/lib/modules/:ro -v /usr/src:/usr/src:ro -v /tmp/tracee:/tmp/tracee aquasec/tracee:latest --trace container=new
</PRE>

### Open another terminal window:
<PRE>
kubectl create deployment whoami --image=training/whoami
kubectl get pods -o wide
curl XXXX:8000
</PRE>
