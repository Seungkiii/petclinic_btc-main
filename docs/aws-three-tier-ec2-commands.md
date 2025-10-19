# AWS 3-Tier EC2 Launch Template Commands

다음 명령어들은 **Amazon Linux 2** 기반의 EC2 인스턴스를 Launch Template의 `User data`에 추가하여 Web (Apache) tier 및 WAS tier를 자동 구성하기 위한 예시입니다.

## 1. Web Tier (Apache HTTP Server)

웹 tier 인스턴트는 Apache를 설치하고 애플리케이션 서버로 Reverse Proxy 하도록 구성합니다. `{APP_SERVER_PRIVATE_IP}` 값을 실제 WAS tier의 프라이빗 IP 혹은 해당 Tier 앞단의 내부 Load Balancer 주소로 치환하세요.

```bash
#!/bin/bash
set -eux

# 패키지 업데이트 및 Apache 설치
yum update -y
yum install -y httpd mod_ssl mod_proxy_html

# Apache 서비스 활성화 및 자동 시작 설정
systemctl enable httpd
systemctl start httpd

# 방화벽을 사용하는 경우 (Amazon Linux 2 기본 firewalld 미사용)
# 필요 시 Security Group에서 80/443 포트 허용

# 가상호스트 설정 생성 (80 포트)
cat <<'APACHE_CONF' > /etc/httpd/conf.d/petclinic.conf
<VirtualHost *:80>
    ServerName petclinic-web

    # 정적 파일 루트 (필요 시 사용자 정의)
    DocumentRoot /var/www/html

    # Reverse Proxy 설정
    ProxyRequests Off
    ProxyPreserveHost On

    <Proxy *>
        Require all granted
    </Proxy>

    ProxyPass        /  http://{APP_SERVER_PRIVATE_IP}:8080/petclinic/
    ProxyPassReverse /  http://{APP_SERVER_PRIVATE_IP}:8080/petclinic/

    ErrorLog /var/log/httpd/petclinic-error.log
    CustomLog /var/log/httpd/petclinic-access.log combined
</VirtualHost>
APACHE_CONF

# Apache 재시작
systemctl restart httpd
```

### 참고 사항
- HTTPS 사용 시 ACM 인증서와 Application Load Balancer를 활용하거나, `mod_ssl`과 자체 인증서를 사용하도록 위 스크립트를 확장하세요.
- SSM Parameter Store 또는 Secrets Manager를 활용하여 `{APP_SERVER_PRIVATE_IP}` 대신 내부 DNS 이름을 주입하면 템플릿 재배포 없이도 변경이 가능합니다.

## 2. WAS Tier (Tomcat + Spring Petclinic)

WAS tier는 OpenJDK 17과 Tomcat 9을 설치하고, 애플리케이션을 WAR 형태로 배포합니다. DB 연결 정보는 `pom.xml`의 MySQL 프로파일을 활용해 환경에 맞게 조정하세요.

```bash
#!/bin/bash
set -eux

# 패키지 업데이트 및 필수 도구 설치
yum update -y
yum install -y java-17-amazon-corretto-devel git

# Tomcat 9 설치 경로 정의
echo "export CATALINA_HOME=/opt/tomcat" >> /etc/profile.d/tomcat.sh
source /etc/profile.d/tomcat.sh

# Tomcat 사용자 및 디렉터리 준비
useradd -m -U -d /opt/tomcat -s /bin/false tomcat
mkdir -p /opt/tomcat
yum install -y wget
TOMCAT_VERSION=9.0.89
wget -q https://dlcdn.apache.org/tomcat/tomcat-9/v${TOMCAT_VERSION}/bin/apache-tomcat-${TOMCAT_VERSION}.tar.gz -O /tmp/tomcat.tar.gz
tar -xf /tmp/tomcat.tar.gz -C /opt/tomcat --strip-components=1
chown -R tomcat:tomcat /opt/tomcat
chmod +x /opt/tomcat/bin/*.sh

# Tomcat 서비스 유닛 생성
cat <<'TOMCAT_SERVICE' > /etc/systemd/system/tomcat.service
[Unit]
Description=Apache Tomcat Application Server
After=network.target

[Service]
Type=forking
User=tomcat
Group=tomcat
Environment="JAVA_HOME=/usr/lib/jvm/java-17-amazon-corretto"
Environment="CATALINA_PID=/opt/tomcat/temp/tomcat.pid"
Environment="CATALINA_HOME=/opt/tomcat"
Environment="CATALINA_BASE=/opt/tomcat"
Environment="CATALINA_OPTS=-Xms512M -Xmx1024M -server -XX:+UseParallelGC"
Environment="JAVA_OPTS=-Djava.awt.headless=true -Djava.security.egd=file:/dev/./urandom"
ExecStart=/opt/tomcat/bin/startup.sh
ExecStop=/opt/tomcat/bin/shutdown.sh
Restart=on-failure

[Install]
WantedBy=multi-user.target
TOMCAT_SERVICE

systemctl daemon-reload
systemctl enable --now tomcat

# 애플리케이션 배포 (Spring Petclinic)
cd /opt
sudo -u tomcat git clone https://github.com/SteveKimbespin/petclinic_btc.git
cd petclinic_btc
sudo -u tomcat ./mvnw -DskipTests package

# 생성된 WAR 파일을 Tomcat webapps에 복사
sudo -u tomcat cp target/petclinic.war /opt/tomcat/webapps/

# Tomcat 재시작
systemctl restart tomcat
```

### 데이터베이스 연동
- Amazon RDS MySQL을 사용하는 경우, `pom.xml`의 `MySQL` 프로파일에 RDS 엔드포인트와 자격 증명을 반영한 뒤 `./mvnw tomcat7:deploy -P MySQL`로 재배포합니다.
- 보안 그룹에서 3306 포트를 WAS tier에서 RDS로만 허용하고, Web ↔ WAS 간에는 8080 포트를 허용하세요.

## 3. 공통 보안 및 운영 권장 사항
- 각 Launch Template에 SSM Agent 설치 명령을 포함해 Systems Manager를 통한 무중단 운영을 지원하세요 (`yum install -y amazon-ssm-agent`).
- CloudWatch Agent 또는 unified CloudWatch Logs 설정을 추가하여 Apache 및 Tomcat 로그를 중앙화합니다.
- EC2 IAM Role에 S3/Parameter Store 접근 권한을 부여해 애플리케이션 설정을 안전하게 주입합니다.
- Auto Scaling Group과 조합 시, Launch Template 업데이트만으로 일관된 배포가 가능합니다.
