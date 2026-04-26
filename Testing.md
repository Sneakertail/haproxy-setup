# Testing

## Sonar server

```sh
sudo apt update
sudo apt install -y openjdk-17-jdk unzip wget

# Optional: Set JAVA_HOME explicitly
sudo nano /etc/profile.d/java.sh
---
export JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
export PATH=$JAVA_HOME/bin:$PATH
---
source /etc/profile.d/java.sh

sudo nano /etc/sysctl.conf
---
vm.max_map_count=262144
fs.file-max=65536
---
sudo sysctl -p

cd /opt
sudo wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-10.4.1.88267.zip

sudo unzip sonarqube-*.zip
sudo mv sonarqube-*/ sonarqube
sudo rm sonarqube-*.zip

sudo useradd -r -M -d /opt/sonarqube -s /bin/bash sonar
sudo chown -R sonar:sonar /opt/sonarqube

sudo nano /etc/systemd/system/sonarqube.service
---
[Unit]
Description=SonarQube service
After=syslog.target network.target

[Service]
Type=forking
ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop
User=sonar
Group=sonar
Restart=always
LimitNOFILE=65536
LimitNPROC=4096

[Install]
WantedBy=multi-user.target
---

sudo systemctl daemon-reload
sudo systemctl enable sonarqube
sudo systemctl start sonarqube

sudo systemctl status sonarqube
```
