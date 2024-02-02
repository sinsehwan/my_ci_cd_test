# my_ci_cd_test
CI/CD 테스트

# CI 절차
    - github 사이트 이동 -> 현 프로젝트로 이동
    - Settings > Security > Secrets and variable > Actions
    - new Repository 버튼 클릭
        - git에 노출되면 안되는 정보들
        - 환경변수(노출되면 안되는 key정보, DB정보 ...)
        - AWS 계정관리 IAM에서 계정별로 access ID/KEY 발급받아서 등록 -> 외부에서 AWS 엑세스 가능
        - action에 의해 .env등 설정 파일로 이관
            - 소스코드상에는 없다.
        - 등록(필요한 만큼, 수시로)
            - 이름, 값

    - gitaction에서 작동한 내용 기술
        - deploy.yml
        - 위치
            - 프로젝트루트 /.github/workflows/deploy.yml
            - gitaction을 작동시키는 이벤트(명령, 트리거)는 
            서브 브런치일 경우 아래처럼 추가 가능
            ```
                pull_request:
                    branches:   [dev]

            ```
        
        - jobs > steps > name => 검증할 내용들을 추가할 수 있다.
            - package.json 추가
                - "build":"babel routes -d build"
                    - 개발된 소스를 표준으로 변환
                    - npm install --save-dev babel-cli


# CD 절차
  - step 1
    - 빌드한 소스코드, 리소스, 기타 설정파일 => 압축(zip)

  - step 2
    - aws 접속
      - IAM 계정상 엑세스키, 엑세스 ID가 gitaction 시크릿 변수에 등록되어 있어야 한다.
      - 계정생성 => 키/아이디 발급 => 등록
      - 분실, 노출 금지

  - step 3
    - 압축(zip)한 리소스 => s3 업로드


  - step 4
    - AWS codeDeploy 서비스 절차에 따라 배포 시작

  - AWS 세팅
    - S3 진입
    - 버킷 만들기 클릭
      - r-deploy-bucket
    - EC2에 역할 부여
      - IAM 진입
        - 엑세스 관리 > 역할 > 역할생성
          - AWS 서비스
            - 사용사례 > EC2
            - 권한추가
              - AmazonS3FullAccess 체크
              - AWSCodeDeployFullAccess 체크
              - 다음
            - 이름 지정, 검토 및 생성
              - 역할이름 - 역할생성

      - EC2에 위에서 만든 역할을 보안에 적용
        - 인스턴스 선택
        - 작업 - 보안 - IAM 역할 수정
        - IAM 역할 선택하고 업데이트

    - CodeDeploy 역할 부여
      - IAM 진입
        - 엑세스 관리 > 역할 > 역할생성
          - AWS 서비스
            - 사용사례 > CodeDeploy
            - 권한추가
              - 다음
            - 이름 지정, 검토 및 생성
              - 역할이름 - 역할생성
      - CodeDeploy 진입
        - 애플리케이션 생성
          - ra-code-deploy
          - EC2/온프레미스
        - 배포그룹생성
          - dev
          - 서비스 역할 : r-code-deploy
        - 환경구성
          - Amazon EC2 인스턴스
            - 키 : name
            - 값 : 인스턴스 지정 : aws-cloud9-node-...
            - 로드벨런서 체크 풀고
            - 배포그룹생성 

        - IAM 사용자 추가
            - 목적 : 엑세스키, ID 발급
            - IAM > 엑세스관리 > 사용자 > 사용자 추가
                - 이름 : user-cicd
                - 다음
                - 권한옵션
                    - AWSS3FullAccess
                    - AWSCodeDeployFullAccess
                    - 다음
                - 사용자 생성
                    - 해당 유저로 진입
                        - 액세스 키 만들기
                        - 기타 > 다음
                        - 태그
                            - accesss-cicd
                                - 액세스 키 : AKIA...
                                - 비밀 액세스 키 : OrSt5...
                                - 위의 2개 값을 가지고 github에 환경변수 등록

# CD - EC2에서 수행할 작업
    - sudo apt install awscli
    - sudo aws configure
        - ID, 시크릿키, 리전, 형식(json) 입력
    - codeDeploy agent
        - wget https://aws-codedeploy-ap-northeast-2.s3.amazonaws.com/latest/install
        - chmod +x ./install
        - sudo apt-get install ruby
        - sudo ./install auto
        - sudo service codedeploy-agent status
            - active (running) : 에이전트 작동중
    - ec2 서버가 재가동했을 때 자동으로 에이전트 가동되게 설정(옵션)
        - sudo nano /etc/init.d/codedeploy-startup.sh
            ```
                #!/bin
                sudo service codedeploy-agent restart
            ```
            - 저장 후 nano 종료 (ctrl + x, y 엔터)
        - sudo chmod +x /etc/init.d/codedeploy-startup.sh

# CD-소스파일
    - appspec.yml 파일 생성(루트)
        - codeDeploy 배포하는 애플리케이션에 대한 사양정리
