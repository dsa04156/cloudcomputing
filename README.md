### 실습 7.1: AWS SDK for Python을 사용하여 Lambda 함수 생성하기

#### 실습 개요 및 목표
이 실습에서는 AWS SDK for Python(boto3)을 사용하여 AWS Lambda 함수를 생성합니다. 이전에 Amazon API Gateway 실습에서 만든 REST API 호출이 이 함수를 실행하게 됩니다. Lambda 함수 중 하나는 Amazon DynamoDB 데이터베이스 테이블 검색 또는 인덱스 검색을 수행합니다. 다른 Lambda 함수는 나중에 Amazon Cognito를 구현할 때 개선할 표준 승인 메시지를 반환합니다.

이 실습을 완료하면 다음 작업을 수행할 수 있습니다:

- DynamoDB 데이터베이스 테이블을 쿼리하는 Lambda 함수를 생성합니다.
- Lambda 함수에 DynamoDB의 데이터를 읽을 수 있는 충분한 권한을 부여합니다.
- Amazon API Gateway를 사용해 REST API 메서드를 구성하여 Lambda 함수를 호출합니다.

#### 소요 시간
이 실습을 완료하는 데 약 90분이 소요됩니다.

#### AWS 서비스 제한 사항
이 실습 환경에서는 실습 지침을 완료하는 데 필요한 서비스 및 작업만 접근할 수 있습니다. 다른 서비스에 접근하거나 실습에 명시되지 않은 작업을 수행하려고 하면 오류가 발생할 수 있습니다.

#### 시나리오
카페는 웹사이트가 데이터베이스에 저장된 데이터를 동적으로 접근할 수 있도록 하기를 원합니다. Sofía는 이 목표를 향해 꾸준히 작업을 진행하고 있습니다.

이전 실습에서는 Sofía 역할을 맡아 DynamoDB 데이터베이스를 생성했습니다. 해당 테이블에는 카페 메뉴 세부 사항이 포함되어 있으며, 인덱스에는 특별 메뉴로 표시된 항목이 저장됩니다. 또 다른 실습에서는 웹사이트가 REST API 호출을 통해 모의 데이터를 수신할 수 있도록 API를 생성했습니다.

이 실습에서는 다시 Sofía의 역할을 맡아 모의 엔드포인트를 실제 기능을 하는 엔드포인트로 대체하여 웹 애플리케이션이 데이터베이스에 연결할 수 있도록 할 것입니다. Lambda를 사용해 GET API와 DynamoDB에 저장된 데이터를 연결하는 브리지를 생성합니다. 또한, POST API 호출에서는 Lambda가 업데이트된 승인 메시지를 반환하도록 설정합니다.

#### AWS Management Console에 접근하기
상단의 **Start Lab**을 선택하여 실습을 시작합니다. **Start Lab** 패널이 열리고, 실습 상태가 표시됩니다.

**팁:** 실습을 완료하는 데 시간이 더 필요한 경우 **Start Lab** 버튼을 다시 눌러 환경의 타이머를 재시작할 수 있습니다.

- **Lab status: ready** 메시지가 표시될 때까지 기다린 후, 패널의 **X** 버튼을 눌러 **Start Lab** 패널을 닫습니다.
- 상단의 **AWS**를 선택합니다. AWS Management Console이 새로운 브라우저 탭에서 열리며 자동으로 로그인됩니다.

**팁:** 새로운 브라우저 탭이 열리지 않는 경우, 브라우저 상단에 팝업 차단 알림이 표시될 수 있습니다. 이 알림을 클릭하여 팝업을 허용하세요.

AWS Management Console 탭을 실습 지침과 함께 볼 수 있도록 배열하세요. 이렇게 하면 실습 단계를 더 쉽게 따라갈 수 있습니다.

**팁:** 실습 지침을 전체 화면에 표시하고 싶다면, 브라우저의 오른쪽 상단에서 **Terminal** 체크박스를 해제하여 터미널을 숨길 수 있습니다.

---
### Task 1: 개발 환경 구성

#### 이 첫 번째 작업에서는 AWS Cloud9 환경을 구성하여 Lambda 함수를 생성할 수 있도록 합니다.

1. **AWS Cloud9 통합 개발 환경(IDE)에 연결합니다.**
   - **Services** 메뉴에서 **Cloud9**을 검색하여 선택합니다.
   - 기존 IDE, **Cloud9 Instance**가 표시됩니다.
   - 해당 IDE에서 **Open**을 선택합니다.
   - AWS Cloud9 IDE가 새 브라우저 탭에서 로드됩니다.

2. **실습에 필요한 파일을 다운로드하고 압축을 해제합니다.**
   - 같은 터미널에서 다음 명령어를 실행합니다:
     ```bash
     wget https://aws-tc-largeobjects.s3.us-west-2.amazonaws.com/CUR-TF-200-ACCDEV-2-91558/05-lab-lambda/code.zip -P /home/ec2-user/environment
     ```
   - AWS Cloud9 인스턴스로 `code.zip` 파일이 다운로드됩니다. 왼쪽 탐색 창에 파일이 표시됩니다.
   - 파일을 압축 해제합니다:
     ```bash
     unzip code.zip
     ```

3. **이전에 완료한 작업을 복원하는 스크립트를 실행합니다.**
   - 스크립트는 Amazon S3 버킷에 카페 웹사이트 코드를 업로드하고, 버킷 정책을 설정하며, DynamoDB 테이블을 생성하고 데이터를 채워 넣습니다. 또한 API Gateway에서 생성한 REST API도 다시 생성합니다.
   - 스크립트의 권한을 설정하고 실행하려면 다음 명령어를 실행합니다:
     ```bash
     chmod +x ./resources/setup.sh && ./resources/setup.sh
     ```
   - IP 주소를 입력하라는 메시지가 표시되면, https://whatismyipaddress.com 에서 확인한 IPv4 주소를 입력합니다.

4. **Python SDK 설치를 확인합니다.**
   - AWS Cloud9 터미널에서 다음 명령어를 실행합니다:
     ```bash
     pip show boto3
     ```

5. **스크립트가 생성한 리소스를 확인합니다.**
   - **Amazon S3 버킷**에 카페 웹사이트 파일이 호스팅되고 있는지 확인:
     - S3 콘솔에서 버킷 이름을 선택하고 **index.html**의 Object URL을 복사하여 새 브라우저 탭에서 로드합니다.
     - 카페 웹사이트가 표시됩니다.

   - **DynamoDB 테이블**에 메뉴 데이터가 저장되었는지 확인:
     - DynamoDB 콘솔에서 **Tables**를 선택하고 **FoodProducts** 테이블을 선택합니다.
     - **Explore table items**에서 테이블이 데이터로 채워져 있는지 확인합니다.
     - **Indexes** 탭에서 `special_GSI` 인덱스가 생성되었는지 확인합니다.

   - **API Gateway**에 `ProductsApi` REST API가 정의되어 있는지 확인:
     - API Gateway 콘솔에서 `ProductsApi` API를 선택합니다.
     - `/products`와 `/products/on_offer`에 대해 GET 메서드가 있는지 확인합니다.
     - `/create_report`에 대해 POST 및 OPTIONS 메서드가 있는지 확인합니다.

6. **API 호출 URL을 복사합니다.**
   - API Gateway 콘솔의 왼쪽 패널에서 **Stages**를 선택한 다음 **prod** 단계를 선택합니다.
   - 페이지 상단에 표시된 Invoke URL을 복사합니다.

7. **웹사이트의 `config.js` 파일을 업데이트합니다.**
   - AWS Cloud9 IDE에서 **resources/website/config.js** 파일을 엽니다.
   - 2번째 줄에서 `null`을 방금 복사한 Invoke URL 값으로 대체하고, URL을 큰따옴표로 감쌉니다.
     ```javascript
     window.COFFEE_CONFIG = {
         API_GW_BASE_URL_STR: "https://<some-value>.execute-api.us-east-1.amazonaws.com/prod",
         COGNITO_LOGIN_BASE_URL_STR: null
     };
     ```
   - 파일을 저장합니다.

8. **`update_config.py` 스크립트를 업데이트하고 실행합니다.**
   - **python_3/update_config.py** 파일을 열고 `<FMI_1>` 자리 표시자를 S3 버킷 이름으로 대체합니다.
   - 파일을 저장한 후 다음 명령어를 실행하여 스크립트를 실행합니다:
     ```bash
     cd ~/environment/python_3
     python update_config.py
     ```

9. **개발자 콘솔을 열어 최신 카페 웹페이지를 로드합니다.**
   - Amazon S3 콘솔에서 웹사이트 파일을 포함하는 버킷을 선택합니다.
   - **index.html**의 Object URL을 복사하여 새 브라우저 탭에서 로드하고 새로 고침합니다.

   - **Browse Pastries** 섹션에 메뉴 항목이 하나만 표시되는 것을 확인합니다. 이는 더 이상 하드코딩된 데이터가 아닌, 이전 실습에서 설정한 대로 모의 데이터를 반환하고 있음을 나타냅니다.

#### 작업 완료 후 구조도
- 실습 시작 시 계정 리소스는 다음과 같은 구조를 가집니다.
- 실습 완료 후에는 Lambda 함수를 생성하여 API가 이를 호출하도록 구성됩니다.
---


### Task 2: Lambda 함수 생성하여 DynamoDB에서 데이터 가져오기

#### 이번 작업에서는 Lambda 함수를 생성하여 DynamoDB에서 데이터를 가져오는 API 엔드포인트를 구성합니다. 이 함수는 웹사이트에서 발생하는 `/products` GET 요청에 응답합니다.

1. **Python 코드 확인 및 수정**
   - AWS Cloud9 파일 브라우저에서 `python_3/get_all_products_code.py` 파일을 찾아 엽니다.
   - `<FMI_1>` 및 `<FMI_2>` 자리 표시자를 적절한 값으로 바꿔야 합니다. 이 값들은 DynamoDB 콘솔에서 확인할 수 있습니다.
   
   - 코드 분석:
     - 이 코드는 `boto3` 클라이언트를 사용하여 DynamoDB와 상호 작용합니다.
     - DynamoDB에서 데이터를 읽어와 메뉴 데이터를 반환합니다.
     - 또한 테이블 인덱스를 스캔하여 재고가 있는 항목만 필터링합니다.
   
   - 변경 사항을 저장합니다.

2. **로컬에서 코드 실행**
   - AWS Cloud9 터미널에서 다음 명령어를 실행하여 작업 중인 디렉토리로 이동합니다:
     ```bash
     cd ~/environment/python_3
     ```
   - 코드를 로컬에서 실행하여 DynamoDB 데이터가 올바르게 출력되는지 확인합니다:
     ```bash
     python3 get_all_products_code.py
     ```
   - 출력 결과는 JSON 형식으로 반환된 일부 메뉴 항목입니다:
     ```json
     running scan on table
     {'product_item_arr': [{'price_in_cents_int': 295, 'special_int': 1, 'tag_str_arr': ['doughnut', 'on offer'], 'description_str': 'A doughnut with blueberry jelly filling.', 'product_name_str': 'blueberry jelly doughnut', 'product_id_str': 'a455'}, ... }
     ```

3. **조건문 수정 후 코드 테스트**
   - `get_all_products_code.py` 파일에서 `line 12`에 있는 조건문을 다음과 같이 수정합니다:
     ```python
     if offer_path_str is None:
     ```
   - 이 조건문을 수정한 후 다시 코드를 실행합니다:
     ```bash
     python3 get_all_products_code.py
     ```
   - 실행 결과로 `on offer` 항목만 필터링되어 출력됩니다. 이는 의도한 결과입니다.

4. **변경사항 되돌리기**
   - 코드가 정상적으로 작동하는지 확인한 후, 조건문을 원래대로 돌려놓습니다:
     ```python
     if offer_path_str is not None:
     ```
   - 코드 마지막 줄을 주석 처리하여 Lambda 함수 호출을 비활성화합니다:
     ```python
     #print(lambda_handler({}, None))
     ```
   - 변경사항을 저장합니다.

5. **IAM 역할 ARN 확인**
   - Lambda 함수에 사용할 IAM 역할을 확인하고 ARN을 복사합니다:
     - AWS 콘솔에서 **IAM** → **Roles**로 이동합니다.
     - `LambdaAccessToDynamoDB` 역할을 찾아 선택하고, 역할 ARN을 복사합니다.
   
6. **Lambda 함수 래퍼 코드 수정**
   - AWS Cloud9에서 `python_3/get_all_products_wrapper.py` 파일을 열고, `line 5`에서 `<FMI_1>`을 복사한 역할 ARN으로 대체합니다.
   - 파일을 저장합니다.

7. **코드 압축 및 S3 버킷에 업로드**
   - AWS Cloud9 터미널에서 코드가 있는 디렉토리로 이동하여 코드를 `.zip` 형식으로 압축합니다:
     ```bash
     zip get_all_products_code.zip get_all_products_code.py
     ```
   - S3 버킷 이름을 확인한 후, 압축된 파일을 해당 버킷에 업로드합니다:
     ```bash
     aws s3 cp get_all_products_code.zip s3://<bucket-name>
     ```

8. **Lambda 함수 생성**
   - `get_all_products_wrapper.py` 파일을 실행하여 Lambda 함수를 생성합니다:
     ```bash
     python3 get_all_products_wrapper.py
     ```
   - 실행 후 `DONE` 메시지가 출력되면 함수가 성공적으로 생성된 것입니다.

9. **Lambda 함수 테스트**
   - Lambda 콘솔로 이동하여 `get_all_products` 함수를 선택합니다.
   - 코드 창에서 `Test`를 클릭하여 테스트 이벤트를 생성합니다.
     - 이벤트 이름: `Products`
     - 기본값을 사용하여 저장합니다.
     - `Test`를 다시 클릭하여 Lambda 함수가 DynamoDB에서 데이터를 반환하는지 확인합니다.

10. **`onOffer` 이벤트 테스트**
    - `Test` 메뉴에서 `Create new event`를 선택하여 `onOffer`라는 새로운 이벤트를 생성합니다.
    - 다음 JSON을 코드 창에 삽입하여 저장합니다:
      ```json
      {
          "path": "on_offer"
      }
      ```
    - `Test`를 클릭하여 `on offer` 메뉴 항목만 반환되는지 확인합니다.
    - 결과에서 `running scan on index`라는 로그 메시지를 확인할 수 있습니다.

11. **Lambda 함수 결과 확인**
    - Lambda 함수가 정상적으로 작동하는 것을 확인한 후, `/products`와 `/products/on_offer` 경로에 대한 GET 요청을 처리할 준비가 완료됩니다.

#### 작업 완료 후
- Lambda 함수가 DynamoDB와 연동되어 웹사이트에서 상품 목록을 동적으로 가져오는 API로 작동합니다.
--- 

### Task 3: Configuring the REST API to invoke the Lambda function

#### 1. **Python 코드 확인 및 수정**
   - **AWS Cloud9**에서 `python_3/get_all_products_code.py` 파일을 찾아 엽니다.
   - 코드 내 `<FMI_1>` 및 `<FMI_2>` 자리 표시자를 적절한 값으로 교체해야 합니다. 이 값들은 DynamoDB 콘솔에서 확인할 수 있습니다.
   - **코드 분석**:
     - `boto3` 클라이언트를 사용하여 DynamoDB와 상호작용합니다.
     - DynamoDB에서 데이터를 읽어와 메뉴 데이터를 반환합니다.
     - 테이블 인덱스를 스캔하여 재고가 있는 항목만 필터링합니다.
   - 수정 후 변경 사항을 저장합니다.

#### 2. **로컬에서 코드 실행**
   - AWS Cloud9 터미널에서 다음 명령어를 실행하여 작업 중인 디렉토리로 이동합니다:
     ```bash
     cd ~/environment/python_3
     ```
   - 코드를 로컬에서 실행하여 DynamoDB 데이터가 올바르게 출력되는지 확인합니다:
     ```bash
     python3 get_all_products_code.py
     ```
   - 출력 결과는 JSON 형식으로 반환된 일부 메뉴 항목입니다:
     ```json
     running scan on table
     {'product_item_arr': [{'price_in_cents_int': 295, 'special_int': 1, 'tag_str_arr': ['doughnut', 'on offer'], 'description_str': 'A doughnut with blueberry jelly filling.', 'product_name_str': 'blueberry jelly doughnut', 'product_id_str': 'a455'}, ... }
     ```

#### 3. **조건문 수정 후 코드 테스트**
   - `get_all_products_code.py` 파일에서 `line 12`에 있는 조건문을 다음과 같이 수정합니다:
     ```python
     if offer_path_str is None:
     ```
   - 이 조건문을 수정한 후 코드를 다시 실행합니다:
     ```bash
     python3 get_all_products_code.py
     ```
   - 실행 결과로 `on offer` 항목만 필터링되어 출력됩니다. 이는 의도한 결과입니다.

#### 4. **변경사항 되돌리기**
   - 코드가 정상적으로 작동하는지 확인한 후, 조건문을 원래대로 돌려놓습니다:
     ```python
     if offer_path_str is not None:
     ```
   - 코드 마지막 줄을 주석 처리하여 Lambda 함수 호출을 비활성화합니다:
     ```python
     #print(lambda_handler({}, None))
     ```
   - 변경사항을 저장합니다.

#### 5. **IAM 역할 ARN 확인**
   - Lambda 함수에 사용할 IAM 역할을 확인하고 ARN을 복사합니다:
     - AWS 콘솔에서 **IAM** → **Roles**로 이동합니다.
     - `LambdaAccessToDynamoDB` 역할을 찾아 선택하고, 역할 ARN을 복사합니다.

#### 6. **Lambda 함수 래퍼 코드 수정**
   - **AWS Cloud9**에서 `python_3/get_all_products_wrapper.py` 파일을 열고, `line 5`에서 `<FMI_1>`을 복사한 역할 ARN으로 대체합니다.
   - 파일을 저장합니다.

#### 7. **코드 압축 및 S3 버킷에 업로드**
   - AWS Cloud9 터미널에서 코드가 있는 디렉토리로 이동하여 코드를 `.zip` 형식으로 압축합니다:
     ```bash
     zip get_all_products_code.zip get_all_products_code.py
     ```
   - S3 버킷 이름을 확인한 후, 압축된 파일을 해당 버킷에 업로드합니다:
     ```bash
     aws s3 cp get_all_products_code.zip s3://<bucket-name>
     ```

#### 8. **Lambda 함수 생성**
   - `get_all_products_wrapper.py` 파일을 실행하여 Lambda 함수를 생성합니다:
     ```bash
     python3 get_all_products_wrapper.py
     ```
   - 실행 후 `DONE` 메시지가 출력되면 함수가 성공적으로 생성된 것입니다.

#### 9. **Lambda 함수 테스트**
   - Lambda 콘솔로 이동하여 `get_all_products` 함수를 선택합니다.
   - 코드 창에서 `Test`를 클릭하여 테스트 이벤트를 생성합니다.
     - 이벤트 이름: `Products`
     - 기본값을 사용하여 저장합니다.
     - `Test`를 다시 클릭하여 Lambda 함수가 DynamoDB에서 데이터를 반환하는지 확인합니다.

#### 10. **`onOffer` 이벤트 테스트**
   - `Test` 메뉴에서 **Create new event**를 선택하여 `onOffer`라는 새로운 이벤트를 생성합니다.
   - 다음 JSON을 코드 창에 삽입하여 저장합니다:
     ```json
     {
         "path": "on_offer"
     }
     ```
   - `Test`를 클릭하여 `on offer` 메뉴 항목만 반환되는지 확인합니다.
   - 결과에서 `running scan on index`라는 로그 메시지를 확인할 수 있습니다.

#### 11. **Lambda 함수 결과 확인**
   - Lambda 함수가 정상적으로 작동하는 것을 확인한 후, `/products`와 `/products/on_offer` 경로에 대한 GET 요청을 처리할 준비가 완료됩니다.


#### **작업 완료 후**
- Lambda 함수가 DynamoDB와 연동되어 웹사이트에서 상품 목록을 동적으로 가져오는 API로 작동합니다.

---
### Task 4: Creating a Lambda function for report requests

#### 1. **Python 코드 확인 및 수정**
   - **AWS Cloud9**에서 `python_3/create_report_code.py` 파일을 찾아 엽니다.
   - 이 코드는 현재 메시지를 반환하는 기본적인 코드입니다. 향후 리포트를 생성하는 더 유용한 로직을 추가할 예정입니다.
   - 로컬에서 코드를 실행하여 동작을 확인합니다:
     ```bash
     python3 create_report_code.py
     ```
   - 터미널 출력은 다음과 비슷하게 나옵니다:
     ```json
     {'msg_str': 'Report processing, check your phone shortly'}
     ```
   - 이 메시지에서 "Report"가 대문자로 표시됩니다. 이는 웹사이트가 Lambda 함수를 호출하고 있다는 것을 나타냅니다.

#### 2. **마지막 줄 주석 처리**
   - `create_report_code.py` 파일에서 코드의 마지막 줄을 주석 처리합니다:
     ```python
     #print(lambda_handler(None, None))
     ```
   - 이 변경 사항을 저장합니다.

#### 3. **Lambda 함수 래퍼 코드 수정**
   - **AWS Cloud9**에서 `python_3/create_report_wrapper.py` 파일을 엽니다.
   - `line 5`에서 `<FMI_1>`을 **LambdaAccessToDynamoDB 역할 ARN**으로 대체합니다.
     - 역할 ARN은 **IAM 콘솔**에서 복사한 값을 사용합니다.
   - 파일을 저장합니다.

#### 4. **코드 압축 및 S3 버킷에 업로드**
   - **AWS Cloud9** 터미널에서 `create_report_code.py` 파일을 `.zip` 형식으로 압축합니다:
     ```bash
     zip create_report_code.zip create_report_code.py
     ```
   - S3 버킷에 `.zip` 파일을 업로드합니다. `<bucket-name>`을 실제 버킷 이름으로 바꾸어 명령어를 실행합니다:
     ```bash
     aws s3 cp create_report_code.zip s3://<bucket-name>
     ```
   - 업로드가 성공적으로 완료되면, 다음과 같은 메시지가 출력됩니다:
     ```bash
     upload: ./create_report_code.zip to s3://<bucket-name>/create_report_code.zip
     ```

#### 5. **Lambda 함수 생성**
   - `create_report_wrapper.py` 파일을 실행하여 Lambda 함수를 생성합니다:
     ```bash
     python3 create_report_wrapper.py
     ```
   - 출력 메시지로 `DONE`이 표시되면, 함수가 성공적으로 생성된 것입니다.

#### 6. **Lambda 함수 테스트**
   - Lambda 콘솔에서 **create_report** 함수를 선택합니다.
   - 코드 창에서 `Test`를 클릭하여 테스트 이벤트를 생성합니다.
     - 이벤트 이름: `ReportTest`
     - 기본값을 사용하여 저장합니다.
   - `Test`를 다시 클릭하여 Lambda 함수가 정상적으로 동작하는지 확인합니다.
   - 테스트 결과는 다음과 같은 JSON 형식으로 출력됩니다:
     ```json
     {
       "msg_str": "Report processing, check your phone shortly"
     }
     ```
   - 메시지에서 "Report"는 대문자로 표시되어 Lambda 함수가 호출되었음을 확인할 수 있습니다.


#### **작업 완료 후**
- `create_report` Lambda 함수가 웹사이트에서 리포트를 생성하는 POST 요청을 처리하는 API로 작동합니다.
---
### Task 5: Configuring the REST API to invoke the Lambda function to handle reports

#### 1. **기존 POST 메소드 테스트**
   - **API Gateway 콘솔**로 이동합니다.
   - `ProductsApi` API를 선택한 후, `/create_report`에 대한 **POST 메소드**를 선택합니다.
   - 페이지 오른쪽에서 **Mock Endpoint**가 설정되어 있음을 확인합니다.
   - `Test`를 선택하고, 페이지 하단의 `Test`를 클릭하여 테스트합니다.
   - 응답 본문에 **mock 데이터**가 반환되는지 확인합니다. 예시:
     ```json
     {
       "message_str": "report requested, check your phone shortly."
     }
     ```

#### 2. **Mock Endpoint를 Lambda 함수로 교체**
   - **POST 메소드**가 여전히 선택된 상태에서, **Integration Request**를 선택하고 **Edit**를 클릭합니다.
   - 다음과 같이 설정을 변경합니다:
     - **Integration type**: Lambda Function
     - **Lambda Region**: us-east-1
     - **Lambda Function**: create_report
   - `Save`를 선택하여 변경 사항을 저장합니다.
   - 이제 **POST 메소드**가 더 이상 Mock Endpoint를 호출하지 않고, **Lambda 함수**를 호출합니다.

#### 3. **메소드 테스트**
   - **POST 메소드**를 다시 테스트합니다.
   - 반환되는 데이터는 이제 **대문자 "R"**로 시작하는 메시지입니다. 예시:
     ```json
     {
       "msg_str": "Report processing, check your phone shortly"
     }
     ```

#### 4. **API 배포**
   - **Resources** 패널에서 **API root (/)**을 선택합니다.
   - `Deploy API`를 선택합니다.
   - **Deployment stage**에서 `prod`를 선택하고, **Deploy**를 클릭하여 API를 배포합니다.

---

### Task 6: Testing the integration using the café website

#### 1. **카페 웹사이트 로드**
   - 웹사이트 페이지를 새로 고칩니다.
     - **Amazon S3 콘솔**에서 버킷 이름을 선택하고, `index.html` 파일을 선택한 후 **Object URL**을 복사하여 새 브라우저 탭에서 로드합니다.
   - 카페 웹사이트가 정상적으로 표시됩니다. 이 웹사이트는 이제 DynamoDB에 저장된 메뉴 데이터를 액세스합니다.

#### 2. **메뉴 항목 필터링 테스트**
   - 웹사이트에서 **Browse Pastries** 섹션으로 이동하여, 기본적으로 "on offer" 메뉴 항목만 표시되는지 확인합니다.
   - "view all"을 선택하여 더 많은 메뉴 항목이 반환되는지 확인합니다.
   - 모든 항목이 정상적으로 표시되면 **CORS** 설정이 제대로 작동하고 있는 것입니다.

#### 3. **DynamoDB에서 가격 변경 후 웹사이트 확인**
   - 웹사이트에서 좋아하는 "on offer" 메뉴 항목의 가격을 확인합니다.
   - 다른 브라우저 탭에서 **DynamoDB 콘솔**을 열고 **FoodProducts** 테이블 항목을 로드합니다.
   - 메뉴 항목의 `product_name` 하이퍼링크를 클릭하여 해당 레코드를 엽니다.
   - `price_in_cents` 값을 변경하고, 변경 사항을 저장합니다.
   - 카페 웹사이트를 새로 고친 후, 가격 변경이 웹사이트에 반영되었는지 확인합니다.

#### **작업 완료 후**
- **POST** 메소드가 성공적으로 **Lambda 함수**를 호출하도록 설정되었고, 웹사이트에서 **DynamoDB** 데이터를 실시간으로 확인할 수 있습니다.
