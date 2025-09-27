# Spring PetClinic Sample Application

[![Build Status](https://travis-ci.org/spring-petclinic/spring-framework-petclinic.svg?branch=master)](https://travis-ci.org/spring-petclinic/spring-framework-petclinic/) 
[![Quality Gate Status](https://sonarcloud.io/api/project_badges/measure?project=spring-petclinic_spring-framework-petclinic&metric=alert_status)](https://sonarcloud.io/dashboard?id=spring-petclinic_spring-framework-petclinic)
[![Coverage](https://sonarcloud.io/api/project_badges/measure?project=spring-petclinic_spring-framework-petclinic&metric=coverage)](https://sonarcloud.io/dashboard?id=spring-petclinic_spring-framework-petclinic)

GCP Cloud 과제 수행을 위한 Java Spring 으로 개발된 Sample Application 입니다. 

**3-layer architecture** (i.e. presentation --> service --> repository) 로 Tomcat 에 배포하여 2 Tier 로 구성하거나
또는 nginx 등의 Web 서버를 통해서 Tomcat 을 연결하는 3 Tier 구성을 테스트할 수 있습니다. 

## Understanding the Spring Petclinic application with a few diagrams

[See the presentation here](http://fr.slideshare.net/AntoineRey/spring-framework-petclinic-sample-application) (2017 update)

## Running petclinic locally

### Tomcat 설치 및 Start
Tomcat 설치 가이드를 참조하여 Tomcat 설치 후, tomcat-users.xml 에 User 및 Role 추가

[ Ubuntu 18.04 : Tomcat 9 설치하는 방법 ](https://jjeongil.tistory.com/1351)

Tomcat User 및 Role 추가

```
# $TOMCAT_HOME/conf/tomcat-users.xml 파일에 아래 행들을 추가

    <role rolename="manager-script"/>
    <role rolename="manager-gui"/>
    <role rolename="manager-jmx"/>
    <role rolename="manager-status"/>
    <user username="tomcat" password="tomcat" roles="manager-gui,manager-script,manager-status,manager-jmx"/>
```

Tomcat 을 실행 ( 위의 Tomcat 설치 가이드를 통해서 이미 실행되어 있는 경우에는 Skip )

```
$TOMCAT_HOME/bin/catalina.sh start
```

### Tomcat 배포 ( H2 In-memory Database 활용 )
```
git clone https://github.com/SteveKimbespin/petclinic_btc.git 
cd petclinic_btc
./mvnw tomcat7:deploy
```

You can then access petclinic here: [http://localhost:8080/petclinic](http://localhost:8080/petclinic)

<img width="1042" alt="petclinic-screenshot" src="https://cloud.githubusercontent.com/assets/838318/19727082/2aee6d6c-9b8e-11e6-81fe-e889a5ddfded.png">

### 외부에 구성한 MySQL Database 연결 방법

MySQL 을 각 CSP 의 DB Service 로 구성
  - database 명 : petclinic  
  - db user 및 password 설정

MySQL database 접속 설정을 하기 위해, pom.xml 파일에 정의 된 'MySQL' profile 을 아래와 같이 수정후, 재배포(redeploy)한다.
  - jdbc.url 부분에 정의되어 있는 DNS 또는 IP Address 를 연결하고자 하는 MySQL IP 로 변경한다. ( 필요시 database 도 수정)
  - db 접속 User ID 및 Password 수정

```
<properties>
    <jpa.database>MYSQL</jpa.database>
    <jdbc.driverClassName>com.mysql.cj.jdbc.Driver</jdbc.driverClassName>
    <jdbc.url>jdbc:mysql://localhost:3306/petclinic?useUnicode=true</jdbc.url>
    <jdbc.username>petclinic</jdbc.username>
    <jdbc.password>petclinic</jdbc.password>
</properties>
```      

Tomcat 에 재배포

```
# 기존에 배포되어 있는 환경에 MySQL 연결 설정 수정후, 재배포 하는 경우
./mvnw tomcat7:redeploy -P MySQL

# 최초 배포하는 경우
./mvnw tomcat7:deploy -P MySQL
```


You can then access petclinic here: [http://localhost:8080/petclinic](http://localhost:8080/petclinic)





# petclinic_btc-main
# petclinic_btc-main
