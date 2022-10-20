# FY23 MADHacks by LongTermWorkers - Technical Note <!-- omit in TOC -->

<br/>

이 문서는 FY23 MADHacks에 LongTermWorkers 팀이 submit한 Data Mesh 데모의 전체적인 구성 및 수행 절차를 간단히 요약한다.

<br/>

- [1. 개요](#1-개요)
- [2. 필요한 OCI Resource들의 Provision](#2-필요한-oci-resource들의-provision)
    - [2.1. VCN](#21-vcn)
    - [2.2. Compute](#22-compute)
    - [2.3. GGMA (GoldenGate Microservice Architecture)](#23-ggma-goldengate-microservice-architecture)
    - [2.4. GGSA (GoldenGate Stream Analytics)](#24-ggsa-goldengate-stream-analytics)
    - [2.5. Databases](#25-databases)
- [3. Data 준비](#3-data-준비)
    - [3.1. At Marketing DB](#31-at-marketing-db)
    - [3.2. Call Center DB](#32-call-center-db)
    - [3.3. Customer DB](#33-customer-db)
- [4. OML Model 생성 및 Deployment](#4-oml-model-생성-및-deployment)
    - [4.1. OML Model 생성](#41-oml-model-생성)
    - [4.2. OML Model deploy](#42-oml-model-deploy)
    - [4.3. Deployed OML Model Test](#43-deployed-oml-model-test)
- [5. GGMA 셋업](#5-ggma-셋업)
    - [5.1. MySQL 셋업 for OGG](#51-mysql-셋업-for-ogg)
    - [5.2. Source DB credential 생성](#52-source-db-credential-생성)
    - [5.3. Extract 생성](#53-extract-생성)
- [6. GGSA 셋업](#6-ggsa-셋업)
    - [6.1. 각종 Connection 및 Reference 생성](#61-각종-connection-및-reference-생성)
    - [6.2. GG Change Data 생성](#62-gg-change-data-생성)
    - [6.3. GoldenGate Stream 생성](#63-goldengate-stream-생성)
    - [6.4. Pipeline 생성](#64-pipeline-생성)
    - [6.5. 각종 Stage 추가로 Pipeline 채워 나가기](#65-각종-stage-추가로-pipeline-채워-나가기)
        - [6.5.1. Query Stage 추가: Join](#651-query-stage-추가-join)
        - [6.5.2. Pattern 스테이지 추가: ML Scoring](#652-pattern-스테이지-추가-ml-scoring)
        - [6.5.3. PREDICTION 설정 Rule](#653-prediction-설정-rule)
        - [6.5.4. Database Table Target 추가](#654-database-table-target-추가)
        - [6.5.5. Kafka Target 추가](#655-kafka-target-추가)

<br/>

# 1. 개요

<br/>

# 2. 필요한 OCI Resource들의 Provision

## 2.1. VCN

- Name: `ltwVCN`
    - VCN Wizard 이용
    - Secutiry List에 다음과 같은 ingress rule 추가
        - Public
            + TCP 443 from all: GGSA & GGMA
            + TCP 80 from all: GGMA
            + TCP 7801 from all: GGBD in GGSA
            + TCP 7811-7830 from all: remote GG to GGBD in GGSA
        - Private
            + TPC 1521 from all: Oracle DB
            + TCP 3306, 33060 from all: MySQL DB

<br/>

## 2.2. Compute

- Public bastion `ltwbastion` (to access databases in the private subnet)
    - MDS 접속을 위해 mysql client 설치
        - 다음 순서로 설치 (`sudo yum localinstall`)
            - mysql-commercial-common-8.0.30-1.1.el8.x86_64.rpm
            - mysql-commercial-client-plugins-8.0.30-1.1.el8.x86_64.rpm
            - mysql-commercial-libs-8.0.30-1.1.el8.x86_64.rpm
            - mysql-commercial-client-8.0.30-1.1.el8.x86_64.rpm
    - 그리고 MySQL Python Connector 설치

<br/>

## 2.3. GGMA (GoldenGate Microservice Architecture)

- Marketplace image로 provision: Oracle GoldenGate 21c for Non-Oracle (MySQL)
    - `ltwVCN`의 public subnet에 provision
    - Deployment: `Source`

- `oggadmin` password 변경 to "WElcome123__"
    - 맥북 크롬 SSL 문제: "thisisunsafe" trick
    - Service Manager 및 Administration Service for `Source` 양쪽 모두 변경

<br/>

## 2.4. GGSA (GoldenGate Stream Analytics)

- Marketplace image로 provision: Oracle GoldenGate Stream Analytics 19c
    - `ltwVCN`의 public subnet에 provision

- `osaadmin` password 변경 to "WElcome123__"

<br/>

## 2.5. Databases

- Marketing DB (ATP shared): `mktDB`
- Call Center DB (MDS in private): `ccDB`.`ccdb`
- Customer DB (Oracle DBCS in private): `custDB`.`custPDB`

<br/>

# 3. Data 준비

## 3.1. At Marketing DB

- 사용자 셋업

```sql
create user mkt identified by WElcome123__;
grant connect, resource to mkt;
grant unlimited tablespace to mkt;
```

- ML table 셋업
    - 이 테이블을 대상으로 ML model을 만들게 됨

```sql
create table mkt.bank_dataset (
    custid number,
    age number,
    job varchar2(64),
    marital varchar2(64),
    education varchar2(64),
    isdefault varchar2(64),
    balance number,
    housing varchar2(64),
    loan varchar2(64),
    contact varchar2(64),
    lday number,
    lmonth varchar2(64),
    lduration number,
    campaign number,
    pdays number,
    previous number,
    poutcome varchar2(64),
    y number
);
```

- ML table에 데이터 로드
    - 먼저 `bank-full.csv`를 임시로 `admin.bankfull`에 load (`Database Action > Data Load` 이용)
    - 그리고 `admin.bankfull`에서 `mkt.bank_dataset`으로 데이터 로드
        - Unique한 custid 추가
        - `y` 컬럼의 `yes`/`no` 값을 각각 number `1`/`0`으로 변환

```sql
declare
    i number;
begin
    i := 0;
    for src_rec in (select * from bankfull) loop
        i := i + 1;
        insert into mkt.bank_dataset values (
            i,
            src_rec.age,
            src_rec.job,
            src_rec.marital,
            src_rec.education,
            src_rec.isdefault,
            src_rec.balance,
            src_rec.housing,
            src_rec.loan,
            src_rec.contact,
            src_rec.lday,
            src_rec.lmonth,
            src_rec.lduration,
            src_rec.campaign,
            src_rec.pdays,
            src_rec.previous,
            src_rec.poutcome,
            case when src_rec.y = 'yes' then 1 else 0 end
        );
    end loop;
    commit;
end;
/
```

- `bank_dataset` 테이블을 unload

```sql
set pagesize 50000
set linesize 10000
set heading off
set feedback off
set timing off
set trimspool on
set markup csv on delimiter , quote off

-- Call Data Record
spool cdr.dat
select custid, contact, lday, lmonth, lduration
  from bank_dataset
order by dbms_random.value;
spool off

-- Call Data Record History
spool cdr_hist.dat
select custid, campaign, pdays, previous, poutcome
  from bank_dataset;
spool off

-- Customer (Demographic) Information
spool cust_info.dat
select custid, age, job, marital, education, isdefault, balance, housing, loan
  from bank_dataset;
spool off
```

<br/>

## 3.2. Call Center DB

- Database 생성

```sql
create database ccdb;
```

- Table 생성

```sql
use ccdb

-- CDC source
create table cdr (
    CUSTID int primary key,
    CONTACT varchar(64),
    LDAY int,
    LMONTH varchar(64),
    LDURATION int,
    PREDICTION varchar(3)
);

-- Join
create table cdr_hist (
    CUSTID int primary key,
    CAMPAIGN int,
    PDAYS int,
    PREVIOUS int,
    POUTCOME varchar(64)
);
```

> MySQL의 identifier는 case-sensitive함. Column name을 대문자로 기술한 것은 만일을 대비해 모델 feature와 이름을 맞추기 위함.

- `cdr_hist`에 데이터 로드
    - MDS에서는 `load data infile ...` 명령을 사용하지 못함.
    - 다른 방법도 있겠지만 insert code가 어짜피 필요하므로 연습 삼아 아래와 같이 작성함

```python
import time
import mysql.connector

config = {
    "user": "admin",
    "password": "WElcome123__",
    "host": "10.0.1.66",
    "database": "ccdb",
    "raise_on_warnings": True
}

conn = mysql.connector.connect(**config)
cur = conn.cursor(prepared=True)

add_cdr_hist = (
    "INSERT INTO cdr_hist "
    "(CUSTID, CAMPAIGN, PDAYS, PREVIOUS, POUTCOME) "
    "VALUES (?, ?, ?, ?, ?)"
)

with open("cdr_hist.dat", "r") as f:
    while True:
        line = f.readline().strip()
        if not line:
            break
        line_list = line.split(",")
        data_cdr_hist = (
            int(line_list[0]),
            int(line_list[1]),
            int(line_list[2]),
            int(line_list[3]),
            line_list[4]
        )

        cur.execute(add_cdr_hist, data_cdr_hist)

conn.commit()
cur.close()
conn.close()
```

- `cdr` insert code
    - 모든 게 셋업된 후 수행 시점에 이용
    - 좀 더 interactive하게 사용할 수 있도록 수정

```python
import time
import mysql.connector

config = {
    "user": "admin",
    "password": "WElcome123__",
    "host": "10.0.1.66",
    "database": "ccdb",
    "raise_on_warnings": True
}

conn = mysql.connector.connect(**config)
cur = conn.cursor(prepared=True)

add_cdr = (
    "INSERT INTO cdr "
    "(CUSTID, CONTACT, LDAY, LMONTH, LDURATION) "
    "VALUES (?, ?, ?, ?, ?)"
)

auto = False

with open("cdr.dat", "r") as f:
    while True:
        if not auto:
            key = input(
                "Press Enter to insert one row or press 'a' to insert all rows: ")
            if key == "a":
                auto = True
                continue
        else:
            time.sleep(1)
        line = f.readline().strip()
        if not line:
            break
        line_list = line.split(",")
        data_cdr = (
            int(line_list[0]),
            line_list[1],
            int(line_list[2]),
            line_list[3],
            int(line_list[4])
        )

        cur.execute(add_cdr, data_cdr)
        conn.commit()
        print("Inserted a row.")

cur.close()
conn.close()
```

<br/>

## 3.3. Customer DB

- 사용자 셋업

```sql
create user cust identified by WElcome123__;
grant connect, resource to cust;
grant unlimited tablespace to cust;
```

- 테이블 셋업

```sql
create table cust.cust_info (
    custid number not null primary key,
    age number,
    job varchar2(64),
    marital varchar2(64),
    education varchar2(64),
    isdefault varchar2(64),
    balance number,
    housing varchar2(64),
    loan varchar2(64)
);
```

- 데이터 로드: SQL*Loader Express 이용
    - <테이블 명.dat>, 즉 `cust_info.dat`이 있는 디렉토리에서 다음 한 라인만 수행하면 됨

```bash
sqlldr userid=cust/WElcome123__@CUSTPDB table=cust_info
```

- Marketing DB에 최종 Target 테이블 생성

```sql
create table mkt.call_result (
    op_type varchar2(64),
    custid number,
    age number,
    job varchar2(64),
    marital varchar2(64),
    education varchar2(64),
    isdefault varchar2(64),
    balance number,
    housing varchar2(64),
    loan varchar2(64),
    contact varchar2(64),
    lday number,
    lmonth varchar2(64),
    lduration number,
    campaign number,
    pdays number,
    previous number,
    poutcome varchar2(64),
    prediction varchar2(64),
    scoring number
);
```

<br/>

# 4. OML Model 생성 및 Deployment

## 4.1. OML Model 생성

- ADB AutoML UI를 통해 생성
    - Experiment `Bank Marketing ML Experiment`
        - Predict: `Y`
        - Prediction Type: Regression
        - Case ID: `CUSTID`
        - Additional Settings
            - Database Service Level: Medium
        - Start w/ Better Accuracy
    - 생성된 모델 중 가장 좋은 걸 선택하여 rename: `BANK_MKT_GMLR_MODEL` (Generalized Linear Model - Ridge)

> 2022.10 현재 오직 regression 만을 지원

<br/>

## 4.2. OML Model deploy

- URI: `BANK_MKT_GMLR_MODEL` (일단 model과 동일하게 가져감)
- Version: `1.0`
- Namespace: `LTW`
- Check Shared


## 4.3. Deployed OML Model Test

- Authentication token 확보

```bash
# omlserver: from Database Action > Oracle Machine Learning RESTful services
export omlserver=https://yh0olybn5pqce4n-mktdb.adb.ap-seoul-1.oraclecloudapps.com
export username=mkt
export password=WElcome123__

curl -X POST --header 'Content-Type: application/json' --header 'Accept: application/json' -d '{"grant_type":"password", "username":"'${username}'", "password":"'${password}'"}' "${omlserver}/omlusers/api/oauth2/v1/token"

export token='eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJNS1QiLCJ0ZW5hbnRfbmFtZSI6Ik9DSUQxLlRFTkFOQ1kuT0MxLi5BQUFBQUFBQTZNQTdLUTNCU0lGNzZVWlFJRFYyMkNBSlMzRlBFU0dQUU1NU0dYSUhMQkNFTUtLTFJTUUEiLCJkYXRhYmFzZV9uYW1lIjoiTUtUREIiLCJjbG91ZF9kYXRhYmFzZV9uYW1lIjoiT0NJRDEuQVVUT05PTU9VU0RBVEFCQVNFLk9DMS5BUC1TRU9VTC0xLkFOVVdHTEpSVlNFQTdZSUFGNlBPT1JUWEJBNDNVVENFVVNGQlpOUlNQN0FMSTJSREZGSlNONFFBU1NTQSIsInJvbGVzIjoiW3tcInJvbGVcIjpcIkdSQVBIX0RFVkVMT1BFUlwiLFwiY29tbW9uXCI6ZmFsc2V9LHtcInJvbGVcIjpcIk9NTF9ERVZFTE9QRVJcIixcImNvbW1vblwiOmZhbHNlfV0iLCJpc3MiOiJFQUY1NEM4REFDQTE2MDVCRTA1MzQzMTAwMDBBNkI4QSIsImV4cCI6MTY2NjIzNjIyNCwiaWF0IjoxNjY2MjMyNjI0fQ.JVRWfW7thykTRDXwvNKDazuT-ej6WQ_gWlXmjN_GsaUVKljzIJ01zxQUdrNndQTyvJAh-FwHLNmVl_AgtWr7Oma_31yLgiGQnLaC2DD8Z80apTweXBHLv4pZcaVXoLwr3gif_wOHK9AqZDfdW_x96wDE2WzttP5E63BB1YN3ELoTPUUp1SDQ1go75OoYo6rxxpaO2x2dYllNoInGKpnj1-_uG2lzfE83NkPMGPHWSlA8v-4gTsXDbucZT5s-NKoAPdsTz4bnOaNyQhDCnJxLvzrbOhUjh-WWSw71Yxkt6Cad_Gnv5CkdT1Mme6obrf-QisWGl_hvznDon9mjRrzIxQ!wjvgFqSiVPlCiHm4dgJ8ug=='
```

- Score

```bash
curl -X POST "${omlserver}/omlmod/v1/deployment/BANK_MKT_GMLR_MODEL/score" \
--header "Authorization: Bearer ${token}" \
--header 'Content-Type: application/json' \
-d '{
   "inputRecords":[
      {
         "AGE":48,
         "BALANCE":-244,
         "CAMPAIGN":1,
         "CONTACT":"unknown",
         "EDUCATION":"tertiary",
         "HOUSING":"yes",
         "ISDEFAULT":"no",
         "JOB":"management",
         "LDAY":5,
         "LDURATION":253,
         "LMONTH":"may",
         "LOAN":"no",
         "MARITAL":"divorced",
         "PDAYS":-1,
         "POUTCOME":"unknown",
         "PREVIOUS":0
      }
   ]
}'
```

> High Prediction Impacts (in order): `LDURATION`, `POUTCOME`, `LMONTH`, `CONTACT`, `HOUSING`, `JOB`

<br/>

# 5. GGMA 셋업

## 5.1. MySQL 셋업 for OGG

- (Oracle과는 달리) 특별히 할 것은 없음. 다만 OGG user는 충분한 권한을 가져야 함
    - MDS의 경우 OGG user는 그냥 `admin` 사용
        - 새로운 user를 생성하는 경우 충분한 권한을 주는 것이 불가능

<br/>

## 5.2. Source DB credential 생성

- Administration Service > Configuration에서 추가
    - Credential Domain: `LTW`
    - Credential Alias: `CCDB_CRED`
    - Database Server: `10.0.1.66`
    - Port: `3306`
    - Database Name: `ccdb`
    - User ID: `admin`
    - Password: `WElcome123__`

- 생성된 `CCDB_CRED`로 접속하여
    - Checkpoint 테이블 생성: `ccdb.gg_checkpoint`
    - Heartbeat: 그냥 submit

<br/>

## 5.3. Extract 생성

- Administration Service > Overview에서 추가
    - `Change Data Capture Extract` 선택
    - Process Name: `CDR_EXT`
    - Credential Domain: `LTW`
    - Credential Alias: `CCDB_CRED`
    - Trail Name: `XT`

- Parameter

```
EXTRACT CDR_EXT
USERIDALIAS  CCDB_CRED, DOMAIN LTW
EXTTRAIL XT
TABLE ccdb.*;
```

- 시작 후 `insert_cdr.py`를 실행하여 몇건 insert
    - 이것이 `CDR_EXT`의 Statistics에 반영됨을 확인

<br/>

# 6. GGSA 셋업

## 6.1. 각종 Connection 및 Reference 생성

- GG Connection (to GGMA)
    - Name: `ogg21cmysql_Connection`
    - Connection Type: `GoldenGate`
    - Service Manager Host: `152.70.254.36`
    - Service Manager Port: `443`
    - GG Username: `oggadmin`
    - GG Password: `WElcome123__`
    - Is SSL: yes
    - Is GG Marketplace: Yes

- Kafka Connection (to local Kafka)
    - Name: `Local_Kafka_Connection`
    - Connection Type: `Kafka`
    - Zookeepers: `localhost`

- Connection for Generic Database (to MySQL Call Center DB)
    - Name: `ccDB_Connection` 생성
    - database: MySql
    - Jdbc url: `jdbc:mysql://address=(host=10.0.1.66)(port=3306)(user=admin)(password=WElcome123__)/ccdb`

- Connection for Oracle Database (to Customer DB)
    - Name: `custDB_Connection`
    - Connection using: `Service Name`
    - Service name: `custpdb.sub10040646451.ltwvcn.oraclevcn.com`
    - Host name: 10.0.1.139 `custdb.sub10040646451.ltwvcn.oraclevcn.com`
    - Port: `1521`
    - Username/Password: `CUST/WElcome123__`

- Connection for Oracle Database (to Marketing DB)
    - Name: `mktDB_Connection`
    - ATP이므로 wallet 파일 업로드에 의해 설정

- Database Table References 생성
    - `CUST_INFO`
    - `CDR_HIST`

<br/>

## 6.2. GG Change Data 생성

- Name: `CDR_GGCD`
- GG Type: `Change Data`
- Connection: `ogg21cmysql_Connection`
- Deployments: `Source`
- Deployment Username/Password: `oggadmin/WElcome123__`
- GG Extracts: `CDR_EXT`
- Target Trail: `RT`
- Kafka Connection: `Local_Kafka_Connection` (받아서 결국 Kafka에 넘길 것이므로)
- GG Change Data Name: `CDR_REP`
- 최초 stop 상태. 이를 start -> 내부적으로 GGBD를 셋업하고 source GGMA와의 Dist Path가 생성되어 실행

<br/>

## 6.3. GoldenGate Stream 생성

- Name: `CDR_STREAM`
- GG Change Data: `CDR_GGCD`
- Table Name: `ccdb.cdr`
- Infer Shape from Stream을 위해 insert를 몇번 수행할 것

<br/>

## 6.4. Pipeline 생성

- Name: `CDR_PIPELINE`
- Stream: `GG_STREAM`
- 생성/시작 후 새 레코드 입력하여 들어오는지 확인

<br/>

## 6.5. 각종 Stage 추가로 Pipeline 채워 나가기

### 6.5.1. Query Stage 추가: Join

- `JOIN_w_CDR_HIST` 생성
- `JOIN_w_CUST_INFO` 생성

> Redundant한 join 컬럼은 remove

<br/>

### 6.5.2. Pattern 스테이지 추가: ML Scoring

- Name: `BANK_ML_SCORE`
- OML server url: https://adb.ap-seoul-1.oraclecloud.com (from Connection String, not REST endpoint)
- Tenant: `ocid1.tenancy.oc1..aaaaaaaa6ma7kq3bsif76uzqidv22cajs3fpesgpqmmsgxihlbcemkklrsqa`
- OML Services Name: `mktdb` (ADB DB name. 하지만 이게 정해진 rule은 아님)
- Username: `MKT`
- Password: `WElcome123__`
- OML Model: `BANK_MKT_GMLR_MODEL`
- Input Fields: `LDURATION`, `POUTCOME`, `LMONTH`, `CONTACT`, `HOUSING`, `JOB` (주요 feature만 사용)

<br/>

### 6.5.3. PREDICTION 설정 Rule

- Name: `SET_PREDICTION`
- Rule: SCORING이 0.5보다 크면 PREDICTION을 'yes'로, 아니면 'no'로 설정
    - 파이 차트도 하나 추가

<br/>

### 6.5.4. Database Table Target 추가

- Target `CALL_RESULT` 생성  `SET_PREDICTION`에 추가

<br/>

### 6.5.5. Kafka Target 추가

- Query Stage 추가: Filter
    - Name: `FILTER_PROSPECTS`
    - Filter: `PREDICTION = 'yes'`

- Kafka Target
    - Name: `PROSPECTS`
    - Connection: `Local Kafka Connection`
    - Topic Name: `BANK_PROSPECTS`
    - Select Existing Shape: `CALL_RESULT`
    - 그리고 `SET_PREDICTION`에 추가

<br/>
