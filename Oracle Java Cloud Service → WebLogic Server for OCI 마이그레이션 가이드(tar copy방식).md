## ✅ 전제 조건

JCS에서 WebLogic Server for OCI로의 마이그레이션을 하기 위해 아래와 같이 몇 가지 방법론을 오라클은 제공합니다. 고객사 환경 상황에 맞게 적절한 방법을 선택합니다.

![마이그레이션 방법론](images/Pasted%20image%20250610194307.png)


이중 다음 전제 조건이 충족되면 tar 복사 마이그레이션 진행방법으로 고려해볼만 합니다:

- Oracle Java Cloud Service는 Oracle Cloud Infrastructure Database(기존 DBCS가 아님) 또는 자율형 데이터베이스를 사용
- Oracle Java Cloud Service 의 버전은 12.2.1.x 이상
- Oracle Java Cloud Service 에 동일한 VCN을 사용 하고 OCI에 Oracle WebLogic Server를 사용할 계획
- Customer 측의 운영 환경 및 애플리케이션에 대한 관리가 전혀 안되어 있을때 고려
- 데이터베이스 클라우드를 기존 환경 그대로 사용하고 싶을때 고려
---
## 📦 마이그레이션 개요

이 가이드에서는 `tar` 명령어를 사용하여 **Oracle Java Cloud Service 인스턴스**를 **OCI용 Oracle WebLogic Server 도메인**으로 마이그레이션하는 절차를 설명합니다.

> 💡 아래 다이어그램은 Oracle Java Cloud Service tar 복사 방식의 마이그레이션 토폴로지를 보여줍니다. 

![마이그레이션 다이어그램](images/Pasted%20image%2020250610080824.png)
  
---
## 🛠 마이그레이션 절차 (요약)

상세 내용 확인전 마이그레이션 전체 절차를 빠르게 살며보면 다음과 같습니다.

1. **Oracle WebLogic Server 생성**

   - Private 서브넷에서 OCI 12.2.1.4 인스턴스에 대해 Oracle WebLogic Server를 생성

2. **DB 포트 접근성 확인**

   - WebLogic Server for OCI가 Java Cloud Service에서 사용하는 **OCI DB 포트**에 접근 가능한지 확인

3. **SSH 포트 접근성 확인**

   - WebLogic Server for OCI가 Java Cloud Service 인스턴스의 **SSH 포트(22)** 에 접근 가능한지 확인

4. **tar 압축 수행 (Java Cloud Service 측)**

   - 다음 디렉토리를 압축
```
/u01/app/oracle/middleware
/u01/data/domains
/u01/jdk
```  

5. **압축 파일 전송**

   - tar 패키지를 Oracle WebLogic Server for OCI 인스턴스로 이동

6. **필수 콘텐츠 교체**

   - 기존 환경에서 가져온 콘텐츠로 OCI 인스턴스의 대응 파일 교체

7. **설정 값 변경**

   - `hostname`과 `database 연결 문자열`을 OCI 환경에 맞게 수정

8. **(Optional) IDCS 설정**

   - Oracle Identity Cloud Service 사용 시 다음 작업 필요:

     - IDCS 인증 공급자의 **Client ID 및 비밀번호 교체**
     - IDCS 애플리케이션 게이트웨이에서 **보호된 경로 설정**

## 🏁 Migration 전 준비 사항

- <font color="red">두 JRF 웹로직 인스턴스가 동시에 실행되서는 안됨</font>(Oracle Java Cloud Service와 이관할 wls for oci 인스턴스)
  -> 같은 데이터 베이스의 OPSS, MDS, Policy Store 등을 두 인스턴스가 동시 접근할 경우, 상태 불일치 및 손상 될수 있기 때문임.
  -> 정식 시작 전까지 서비스 Listen 제한(외부 액세스 제한, SSH 포워딩으로만 접근), DB 스키마 분리(타겟의 RCU는 임시 또는 복제 DB 스키마 사용(DB export/import or DB Snapshot기반으로 RCU 구성)
  
- JCS의 서비스 인스턴스의 현재 사용 중인 형태와 유사한 Oracle WLS for OCI에 대한 compute를 선택합니다.
  
- 여기서 기존 JCS에서 사용하는 웹로직 라이선스에 맞춰서 WebLogic edition 선택(Standard, Enterprise, Suite edition)
  
- 네트워크에 대한 보안 규칙은 다음 링크 내용을 참고합니다.
  https://docs.oracle.com/en/cloud/paas/weblogic-cloud/jcsmc/configure-security-rules-network.html#GUID-1413EEA2-BDA0-48CB-BA06-E163C6593B95
  
  - 🪏 타겟 wls for oci 구성시 아키텍처 구성 및 포트 내용을 참고해서 구성합니다. 아래는 스크린샷은 프로비저닝 구성 예시입니다.
  
  - admin console 접속이 어려워 고객으로 부터 config.xml 제공 받았을때 포트 및 환경 파악
  ![[Pasted image 20250610175416.png|500]]
 
- 마이그레이션 대상 웹로직 인스턴스의 비밀번호를 <font color="red">JCS에서 사용했던 웹로직 관리자 비밀 번호로 설정</font>해줍니다.

##  🪏마이그레이션 상세

- ### WLS for OCI 인스턴스 생성
  
  - OCI 마켓플레이스에서 JCS에서 사용한 웹로직 버전 및 Edition 선택(Stack방식으로 프로비저닝)
    
  - JCS에서 사용한 버전과 관계없이 WLS for OCI 12.2.1.4 버전의 인스턴스 생성
    
  - 웹로직 Admin서버, Managed 서버 외부 포트를 일치 시킴
    
  - JCS가 OTD를 사용하기 때문에 OCI에서도 로드 밸런서를 프로비저닝 해줘야 함
    
  - Oracle Java Cloud Service 인스턴스에서 Oracle Identity Cloud Service(IDCS)를 사용한 경우 로드 밸런서 프로비저닝 및 Identity Cloud Service를 사용하여 인증 사용을 선택
    
  - 모든 Oracle Java Cloud Service 가 JRF를 사용하지만, Oracle Java Cloud Service 인스턴스 에서 마이그레이션된 도메인은 JRF 옵션으로 생성 된 새 스키마가 아닌 기존 JRF 데이터베이스 스키마를 가리키므로 JRF로 제공 옵션 을 선택할 필요가 없음(필요 파일은 tar copy로 넘겨지고 DB는 기존것을 사용하므로)
    
  - JCS에서 사용했던 볼륨 크기를 고려하여 타겟도 동일한 볼륨 설정
    
  -  아래 WLS for OCI 프로비저닝 스크린샷은 예시입니다.
     ![[Pasted image 20250610174753.png]]

- ### JCS, WLS for OCI의 웹로직 서버 프로세스 중지
  
  - OCI용 Oracle WebLogic Server 에서 admin 노드 및 managed 서버 노드에 아래와 같은 명령을 실행합니다. 노드가 하나면 아래 명령어는 한번이면 됩니다.
```
sudo su - oracle
/opt/scripts/restart_domain.sh -o stop
# run jps to confirm that there are no more processes are running. If there are run kill -9 on each.
jps
```  

- JCS에서 Oracle WebLogic 서버 관리 콘솔을 사용하여 Oracle Java Cloud Service 인스턴스 에서 모든 WebLogic 서버를 중지합니다.
  - Oracle은 마이그레이션 프로세스를 시작하기 전에 Oracle Java Cloud Service 의 WebLogic 서버를 중지할 것을 권장합니다. 
    -> JCS가  운영중인 상황에서는 적절한 다운타임 시간을 정해서 tar copy한 후 JCS를 다시 구동 후에 타겟 웹로직 인스턴스 작업을 진행해도 됩니다. 
    

- ### 도메인 및 바이너리 교체
  
  - OCI 인스턴스의 Oracle WebLogic Server 에서 도메인과 바이너리를 교체합니다.
  
  1) Oracle Java Cloud Service 인스턴스 에서 WebLogic 관리 서버를 실행하는 VM에 액세스 하고 다음 명령을 실행하여 WebLogic 도메인과 바이너리를 tar 압축해 놓습니다.
```
sudo su - oracle
cd /u01/data/domains
tar cvf /tmp/domain.tar .
cd /u01/app/
tar cvf /tmp/mw.tar .
cd /u01/jdk/
tar cvf /tmp/jdk.tar .
```  
  
  2) Oracle 사용자로 로그인 하고 Oracle JCS 인스턴스의 각 추가 VM에 ssh로 접속 하여 WebLogic 도메인 콘텐츠를 tar 압축해 놓습니다.
```
cd /u01/data/domains
tar cvf /tmp/domain.tar .
```  
  
  3) SCP copy를 위해서 JCS 인스턴스 에서 Oracle WebLogic Server for OCI 인스턴스로의 ssh 액세스를 설정합니다.
    a) JCS 인스턴스에서 사용자 홈 에 있는 개인 키로부터 공개 키를 생성
```
# Must be run as oracle user. If you are the opc user, run 'sudo su - oracle' first.
ssh-keygen -y -f ~/.ssh/id_rsa
```
 
    b) 출력을 저장합니다.
    c) opc 사용자로 Oracle WebLogic Server for OCI 인스턴스 에 액세스
    d) WLS for OCI 인스턴스에 있는 각 VM에서 `authorized_keys`사용자에 대한 출력을 추가
    
```
sudo su - oracle
vi ~/.ssh/authorized_keys
# add the public key to this file.
```
  
	e) WLS for OCI 인스턴스에 있는 각 VM에서 다음 명령을 실행하여 현재 도메인, 미들웨어 홈, JDK 홈에서 콘텐츠를 이동

```
sudo su - oracle
mkdir /tmp/domain_bak
mv /u01/data/domains/* /tmp/domain_bak/
mkdir /tmp/mw_bak
mv /u01/app/* /tmp/mw_bak/
```
    
  4) WLS for OCI 인스턴스에 `domain.tar`, `mw.tar`, jdk.tar 파일을 복사
    a) 사용자로 Oracle Java Cloud Service 인스턴스 에 액세스 하고 scp를 실행하여 각 VM의 tar 파일을 OCI 인스턴스의 Oracle WebLogic Server`opc` 에 있는 해당 VM으로 복사
    다음 예에서는 OCI 인스턴스에 대한 Oracle WebLogic Server 에 IP 주소가 있는 단일 VM이 있습니다 `10.1.1.1`

```
sudo su - oracle
scp -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no /tmp/domain.tar 10.1.1.1:/tmp/
scp -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no /tmp/mw.tar 10.1.1.1:/tmp/
scp -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no /tmp/jdk.tar 10.1.1.1:/tmp/
```
    
    b)WLS for OCI 인스턴스에 opc 사용자로 ssh 접속후 각 VM에서 다음을 실행합니다.
    
```
sudo su - oracle
tar xvf /tmp/domain.tar -C /u01/data/domains/
tar xvf /tmp/mw.tar -C /u01/app/
mkdir /u01/app/oracle/jdk
tar xvf /tmp/jdk.tar -C /u01/app/oracle/jdk/
```


- ### 도메인 컨텐츠 업데이트
  
  - Cloning 스크립트를 사용하여 도메인 컨텐츠를 업데이트를 하려면 다음 단계를 따라합니다.
  
  1) Oracle 사용자로 WLS for OCI 에 있는 VM에서 다음 명령을 실행합니다.
     
```
sudo su - oracle
python3 /opt/scripts/cloning/update_service_config.py metadata 1
```
   이를 통해 WLS for OCI의 재시작 및 확장 스크립트에 사용되는 메타데이터에 올바른 정보가 포함되어 있는지 확인할 수 있습니다.
   
> 💡 메타 데이터 다음의 숫자는 0과 1로 지정 가능한데 0: wls for oci, 1: jcs로 이해하면 됩니다. Oracle Java Cloud Service는 도메인 객체를 8자로 제한합니다. 그러나 OCI용 Oracle WebLogic Server에는 이러한 제한이 없기때문에 0(wls for oci)로 테스트 할 경우 도메인네임이 짤리는 경우가 있음. 관련 문제를 위해선 cloning 스크립트를 약간 수정해줘야 함.

  2) 도메인 Config 파일의 호스트 이름을 OCI 인스턴스의 Oracle WebLogic Server 에 있는 VM의 호스트 이름으로 변경, Oracle 사용자로 WLS for OCI인스턴스에 있는 각 VM에서 다음 명령을 실행

```
sudo su - oracle
python3 /opt/scripts/cloning/update_service_config.py hostname 1
```

  3) 데모 ID 인증서에 잘못된 호스트 이름이 있는 경우 사용자로서 OCI 인스턴스 의 Oracle WebLogic Server`oracle` 에 있는 각 VM에서 다음 명령을 실행

```
sudo su - oracle
/opt/scripts/cloning/generate_demo_certs.sh
```

  4) Compute 인스턴스 재부팅 시 서버가 다시 시작되도록 마커 파일이 있는지 확인

```
touch /u01/data/domains/<jcs_domain_name>/provCompletedMarker
```

다음을 `<jcs domain name>`실행하여 찾을 수 있습니다.

```
ls /u01/data/domains/ | grep _domain
```


- ### WebLogic Server 프로세스 시작
  
  - Oracle Java Cloud Service 인스턴스 에서 WebLogic 서버를 중지하지 않았다면 지금 바로 중지하세요 . Oracle은 JRF를 사용하여 두 개의 WebLogic 도메인을 동시에 실행하는 것을 지원하지 않습니다. WebLogic Server 관리 콘솔을 사용하여 모든 서버를 중지하세요.
    
  - WLS for OCI 인스턴스에서 다음 단계를 완료
  
  1) Oracle 사용자로 Admin, Managed Server node에서 다음 명령을 실행

```
sudo su - oracle
/opt/scripts/restart_domain.sh -o start
# run jps to confirm that the processes are running.
jps
```


- ### IDCS 마이그레이션(Optional)
  
  - OCI를 위해 Oracle WebLogic Server 에서 생성한 Oracle Identity Cloud Service(IDCS) 기밀 애플리케이션은 Oracle Java Cloud Service를 위해 생성한 IDCS 애플리케이션을 대체해야함.
  
    1) IDCS 인증 공급자에서 클라이언트와 비밀번호를 업데이트
       a)IDCS 콘솔에서 탐색 서랍을 확장한 다음 응용 프로그램을 클릭
       b)도메인과 연결된 엔터프라이즈 애플리케이션을 클릭
       c)해당 애플리케이션의 이름은 다음과 같습니다.`<stack>_confidential_idcs_app_<timestamp>`
       예: `myweblogic_confidential_idcs_app_2019-08-01T01:02:01.123456`.
       d)`Client ID`및 값을 검색합니다 `Client Secret`.
       e)WebLogic Server 관리 콘솔을 사용하여 IDCS 인증 공급자의 클라이언트와 비밀번호를 업데이트
        i) WebLogic Server 관리 콘솔에 로그인
        ii) 보안 영역 으로 이동
        iii) 영역을 선택하세요. 기본값은 .입니다 `myrealm`
        iv) Provider 탭 클릭
        v) IDCSIntegrator를 선택
        vi) Provider별 탭을 클릭
        vii) 잠금 및 편집 클릭
        viii)Client ID와 Client 비밀번호를 IDCS 기밀 애플리케이션에서 검색한 값으로 변경
        
    2) Cloudgate 역할을 추가
    -  Oracle WebLogic Server for OCI 인스턴스 용으로 생성된 IDCS 기밀 애플리케이션은 Oracle Java Cloud Service 에 설정된 것보다 더 제한적인 기본 역할인 Authenticator Client를 설정합니다 . Enterprise Manager Fusion Middleware Control 콘솔에서 동일한 수준의 권한을 유지하려면 클라이언트 애플리케이션을 업데이트하여 Cloud Gate 역할을 추가해야 함. 아래 내용 참고
      https://docs.oracle.com/en/cloud/paas/weblogic-cloud/user/integrate-opss-user-and-group-apis-identity-cloud-service.html
      
    3) WLS for OCI에서 WebLogic 서버 프로세스를 다시 시작
       a) opc사용자로 인스턴스의 admin, managed server VM에 로그인하고 다음 명령을 실행

```
sudo su - oracle
/opt/scripts/restart_domain.sh
# run jps to confirm that the processes are running.
jps
```

    4) 보호된 경로를 이동
       IDCS로 보호할 서버 컨텍스트 경로를 설정한 경우 해당 경로를 마이그레이션해야 함.
       아래 내용 참조
       https://docs.oracle.com/en/cloud/paas/weblogic-cloud/jcswt/migrate-oracle-identity-cloud-service-roles-and-policies-wlc-wdt-migration.html


## 💡참고 사항

 - 현재 모든 jcs서비스가 서비스 종료되어 내부 테스트가 불가능하여 wls for oci로 테스트시 비슷한 환경을 만들어야 했음(jrf, non-jrf 테스트)
 - 오라클에서 제공되는 cloning 스크립트는 jcs기준의 도메인 이름 8자 제한에따라 아래처럼 스크립트를 수정해줘야 함 
```
[opc@skche-wls-0 cloning]$ vi update_service_config.py

392 라인을 아래처럼 수정
adminserver_name=domain_name[0:8] + "_adminserver"
-> adminserver_name = domain_name.replace("_domain", "") + "_adminserver"

---

412 라인을 아래처럼 수정
managed_server_name=domain_name[0:8] + '_server_' + str(host_index + 1)
-> managed_server_name=domain_name.replace("_domain", "") + "_server_" + str(host_index + 1)

---

560 라인을 아래처럼 수정
jcs_resource_prefix=domain_name[0:8]
-> jcs_resource_prefix=domain_name.replace("_domain", "")

```

 - 운영 환경의 wls for oci의 데이터소스의 패스워드는 암호화되어 있어 고객이 패스워드 관리를 하지 않았을 시 복호화 방법을 고객에게 알려줘야 할 수 있음
   
 - 애플케이션 복잡도가 낮은 경우 앱 배포 방식을 선택해서 Non-JRF 웹로직을 선택할 수 있어 데이터베이스 종속성 부분에 대한 부담을 줄 일수 있음
   
 - OTD에 있는 JCS ssl 인증서는 오라클에서 자동 발급 관리하였으나 WLS for OCI에서는 도메인 등록 및 SSL 인증서 등록을 Customer가 챙겨야 함. 마이그레이션 전 고객과 미리 협의해 두어야 좋음
   
 - JCS를 Oracle Fusion Apps와의 연동을 위해 SaaS extension(JCS-sx서비스는 아님)형태로 사용하면 IDCS SSO 연동 부분을 체크해야 하나 사이트에서 해당 부분을 사용하지 않고 도메인 액세스 제어 부분만 콘트롤 할 수 있음.(특정 케이스인 경우) 아래와 같이 CORS 설정 부분에 접근 허용 IP, 도메인 등록이 필요
   ![[Pasted image 20250611092120.png]]
   https://docs.oracle.com/en/cloud/saas/human-resources/25b/farws/Configure_for_CORS.html
