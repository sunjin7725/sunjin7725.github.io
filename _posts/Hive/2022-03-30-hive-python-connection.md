---
title: Hive-Python Connection
categories: [hive]
tags: [hive, python, odbc]
comments : true
---

# Hive Python Connect

# Hive odbc 드라이버 설치

hive odbc driver 설치 홈페이지

([https://www.cloudera.com/downloads/connectors/hive/odbc/2-6-1.html](https://www.cloudera.com/downloads/connectors/hive/odbc/2-6-1.html))

💥 impala를 사용할 경우에는 ([https://www.cloudera.com/downloads/connectors/impala/odbc/2-6-11.html](https://www.cloudera.com/downloads/connectors/impala/odbc/2-6-11.html))

# Hive odbc 설정

## Window

1. odbc 드라이버 설정으로 들어간다.
    
    ![Untitled](/assets/img/post/odbc-1.png)
    
2. 추가를 선택한다.
    
    ![Untitled](/assets/img/post/odbc-2.png)
    
3. Cloudera ODBC Driver for Apache Hive 선택

![Untitled](/assets/img/post/odbc-3.png)

1. 사용하는 Hive 설정에 맞추어 정보들을 넣어준다.

![Untitled](/assets/img/post/odbc-4.png)

## Linux

RedHat 계열의 OS 위주로 설명하겠습니다.

데비안 계열은 일부 설치 파일만 다르니 검색해서 확인하시면 될것 같습니다.

Reference: [https://www.cdata.com/kb/tech/hive-odbc-python-linux.rst](https://www.cdata.com/kb/tech/hive-odbc-python-linux.rst)

1. unixODBC를 설치한다.

```bash
# in Red Hat
sudo yum install unixODBC unixODBC-devel

# in Debian
sudo apt-get install unixODBC unixODBC-dev
```

1. unixODBC 설치 확인

```bash
odbcinst -j

#################
unixODBC 2.3.1
DRIVERS............: /etc/odbcinst.ini
SYSTEM DATA SOURCES: /etc/odbc.ini
FILE DATA SOURCES..: /etc/ODBCDataSources
USER DATA SOURCES..: /root/.odbc.ini
SQLULEN Size.......: 8
SQLLEN Size........: 8
SQLSETPOSIROW Size.: 8
```

1. Hive ODBC 드라이버 설치

[https://www.cloudera.com/downloads/connectors/hive/odbc/2-6-1.html](https://www.cloudera.com/downloads/connectors/hive/odbc/2-6-1.html)

에서 OS에 맞추어서 다운 받기!

```bash
rpm -i /path/to/hive_driver_installer
```

1. /etc/odbc.ini 에 hive 설정하기

```bash
#vi /etc/odbc.ini
[ODBC Data Sources]
hive=Cloudera Hive ODBC Driver

[hive]
Driver=/opt/cloudera/hiveodbc/lib/64/libclouderahiveodbc64.so
HiveServerType=2
Host=$HIVE_HOST
Port=$HIVE_PORT
UID=$HIVE_UID
PWD=$HIVE_PWD
```

1. 잘설정되었는지 확인하자

```bash
isql hive
```

# 추가사항

python 에서 jaydebeapi를 사용하면 jdbc connection 도 지원합니다.

다만 python이 c 기반이다 보니 odbc를 사용함이 올바른(?) 방법으로 이해하고 있습니다.

하지만 하이브나 임팔라 드라이버를 사용하다보니 드라이버 내에서의 문제로 정상적으로 동작하지 않을 때가 있습니다. 모두 참고하시길...