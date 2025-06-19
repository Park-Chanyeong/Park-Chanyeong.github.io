---
layout: single
title: "Terraform 톺아보기"
categories: DataEngineering
tag: [terraform]
search: true
typora-root-url: ../
sidebar:
  nav: "counts"








---



**[**Terraform ][Terraform 톺아보기 ](https://park-chanyeong.github.io)
{: .notice--primary}

# Terraform

------

## 1. Terraform 개요

![Terraform이란?, 실습 (ec2, rds 생성해보기)](https://joojae.com/content/images/2023/11/terraform-logo-1.png)

### 1.1 IaC란?

- Infrastructure as Code (IaC)란 인프라를 코드로 정의하고 관리하는 방식임
- Terraform은 HashiCorp에서 만든 대표적인 IaC 도구로, 멀티 클라우드 환경 지원 (AWS, GCP, Azure 등)
- 코드 기반으로 인프라 상태를 선언하고, 이를 `terraform apply` 명령으로 자동 생성함

### 1.2 Terraform의 특징

- 인프라를 코드로 관리
  - 인프라 구성을 코드 형태로 관리함으로써, 버전 관리, 재사용, 공유가 용이해진다.
- 자동화 및 효율성
  - 반복적인 인프라 작업을 자동화하여 시간과 노력을 절약할 수 있다.
- 클라우드 독립적
  - 여러 클라우드 제공업체에서 동일한 도구를 사용하여 일관된 경험을 제공한다.
- 선언적 구성 언어
  - HCL을 사용하여 인프라의 원하는 최종 상태를 쉽게 기술할 수 있다.
- 변경 전 시뮬레이션(`plan`) → 실제 반영(`apply`) → 정리(`destroy`)의 명확한 실행 흐름

![image (11)](/images/2025-05-30-airflow/image (11).png)

### 1.3 그냥 AWS CDK로 CloudFormation 쓰면 되는 거 아님? 이것도 IaC인데?

- **AWS CloudFormation**은 JSON/YAML이라 가독성이나 반복 구조가 좀 힘듦. 그리고 AWS 외 다른 클라우드는 못 씀.
- **CDK**는 Python/TypeScript로 인프라를 프로그래밍하듯 작성할 수 있어서 익숙한 개발자에겐 편하지만, 결국 내부적으로 CloudFormation으로 바뀌기 때문에 **상태 관리나 복잡도에서 Terraform보단 제한이 있음.**
- 반면 Terraform은 **멀티 클라우드**도 되고, **HCL**이라는 전용 DSL로 구조가 명확하고 선언적임. 인프라 코드에만 집중하기 좋아서 **DevOps 엔지니어 관점에서 더 일관된 경험 제공함.**

즉, 단일 AWS 프로젝트는 CDK도 나쁘진 않지만, **멀티 클라우드, 팀 협업, 모듈화, 재사용성, 실무 자동화 측면에선 Terraform이 좀 더 유연함.**

[AWS CDK에서 Terraform으로](https://tech.inflab.com/202202-aws-cdk-to-terraform/)

### 1.4 그럼 Ansible은 뭐임 이것도 IaC 도구라는데?

둘 다 인프라 관련 자동화 도구긴 한데 **역할이 다름.**

- **Terraform**: AWS 같은 클라우드에서 리소스를 프로비저닝 하는데에 특화됨 (EC2, VPC, RDS 생성 등)
- **Ansible**: 서버 내부 설정을 자동화함 (패키지 설치, 서비스 설정, 구성 변경 등)

예를 들어 **EC2 기반 Spark 클러스터나 Airflow 플랫폼을 구성**한다고 했을 때:

- **Terraform**으로는 VPC, Subnet, Security Group, EC2 인스턴스 수, IAM Role 등 클라우드 리소스 자체를 선언함. (프로비저닝)
- **Ansible**은 그 인스턴스 안에 Airflow, Spark, Java, Python, 설정 파일, 서비스 등록 등을 SSH로 접속해서 자동으로 구성하는 것!
- 예를들어, 마스터 노드에 kafka 바이너리 파일이 있다고 치고 이 Playbook을 실행하면 새 서버에 `kafka` 전용 계정을 만들고, Kafka 실행에 필요한 디렉토리 구조를 만들고, Kafka 바이너리 압축 해제까지 다 해주는 거임.

```yaml
---
- name: install Confluent Kafka
  hosts: kafka
  tasks:
    - name: Add Kafka group
      ansible.builtin.group:
        name: kafka
        gid: 1001
        system: false
      become: yes
# 유저 및 그룹 생성
    - name: Add Kafka user
      ansible.builtin.user:
        name: kafka
        shell: /bin/bash
        uid: 1001
        group: kafka
        groups: ubuntu
        comment: User for Kafka Cluster
      become: yes

    - name: make directory
      ansible.builtin.file:
        path: /engine
        state: directory
      become: yes
      
# Kafka 바이너리를 압축 해제할 경로 생성
    - name: unarchive file
      ansible.builtin.unarchive:
        src: /home/ec2-user/downloads/confluent-community-6.2.14.tar.gz
        dest: /engine
        copy: true
        owner: kafka
        group: kafka
      become: yes
      
# Kafka 바이너리를 압축 해제할 경로 생성
    - name: make data directory
      ansible.builtin.file:
        path: /data
        state: directory
      become: yes

    - name: make data directory
      ansible.builtin.file:
        path: /src
        state: directory
      become: yes
      
# 로그나 segment 파일 저장용 디렉토리

    - name: create log directory
      become: yes
      file:
        path: /log
        owner: kafka
        group: kafka
        state: directory

    - name: make kafka segment directory
      ansible.builtin.file:
        path: /data/kafka-logs
        state: directory
        owner: kafka
        group: kafka
      become: yes
```

요약하자면 Terraform이 **클라우드 기반 인프라를 만드는 도구**라면, Ansible은 **그 위에 서비스나 플랫폼을 자동으로 얹는 도구**임. → **한 줄 요약:** Terraform은 인프라 생성, Ansible은 인프라 구성

- 셸 스크립트보다 **가독성 좋고**, **반복 실행 시 안전함** (필요 없는 작업은 건너뜀)
- 한 번 만든 `Playbook`은 수십 대 서버에 반복적으로 재사용 가능
- 클러스터형 애플리케이션 설정(예: Kafka, Spark 등)에 특히 유용

------

## 2. 기본 파일 구조와 실행 흐름

### 2.1 파일 구성

### terraform 디렉터리 구조

- 기본적으론 이런 구조 (dev, prod, qa 같이 나눠서 각각 구성할 수도 있음)

| 파일명              | 설명                                                |
| ------------------- | --------------------------------------------------- |
| `main.tf`           | 핵심 리소스 정의 (예: EC2, VPC, S3 등)              |
| `variables.tf`      | 외부로부터 받는 변수 정의 (`variable "region" {}`)  |
| `terraform.tfvars`  | 변수에 실제 값 입력 (`region = "ap-northeast-2"`)   |
| `outputs.tf`        | 실행 결과에서 외부로 노출할 값 정의 (예: 퍼블릭 IP) |
| `terraform.tfstate` | 실제 인프라 상태를 저장하는 파일 (자동 생성됨)      |

> *.tf 파일은 순서 상관없이 모두 병합되어 실행됨

### 2.2 명령어 흐름

![image.png](/images/2025-05-30-airflow/image (12).png)

### 1. init

Terraform 프로젝트를 초기화하는 데 사용됨.

```bash
terraform init
```

- init을 하게 될 경우 실행한 경로에 .terraform파일이 생성되며 지정한 Provider에 해당하는 파일을 다운로드 (처음 한번만 해주면 된당)
- terraform에서 사용되는 프로바이더, 모듈 등의 지정된 버전에 맞춰 root module을 구성하는 역할을 수행
- 짤막 팁 ) 프로바이더 종속성을 고정시키는 **.terraform.lock.hcl 파일이 있는데,**

**해당 파일에 명시된 버전으로 terraform init 명령을 실행하고,**

이후 다른 **작업자가 의도적으로 버전을 변경하거나 코드에 명시된 다른 버전으로 변경하려면 terraform init -upgrade 옵션이 붙은 명령어를 실행해야함.**

```yaml
# .terraform.lock.hcl 파일
provider "registry.terraform.io/hashicorp/aws" {
  version = "5.97.0"
  hashes = [
    "h1:rIcRZfPZXOp3lUPM+TVqvO2JTWOqUuzQ7DDQ3wb9q60=",
    "zh:02790ad98b767d8f24d28e8be623f348bcb45590205708334d52de2fb14f5a95",
    "zh:088b4398a161e45762dc28784fcc41c4fa95bd6549cb708b82de577f2d39ffc7",
  ]
}
# 요 위에 해시값은 provider가 변조되거나 바뀌지 않았는지 확인하는 데 사용됨.
```

- **작업자가 의도적으로 버전을 변경하거나 코드에 명시된 다른 버전으로 변경하려면 terraform init -upgrade 옵션이 붙은 명령어를 실행해야됨**

### 2. vaildate

```bash
terraform validate
```

![image (13)](/images/2025-05-30-airflow/image (13).png)

이상이 없으믄 요렇게 나온다 ㅇㅅㅇ

![image (14)](/images/2025-05-30-airflow/image (14).png)

### 2-1. fmt (format) - 대충 Python의 black 친구

작성한 `.tf` 코드의 **포맷(들여쓰기, 줄바꿈 등)을 자동 정렬**해줌.

```bash
terraform fmt
```

- 가독성과 통일성 확보용으로 한번씩 씀.
- 실무에선 커밋 전에 항상 한 번 돌리는 게 관례라고 함..
- `terraform fmt -recursive` 옵션으로 하위 디렉토리까지 포맷 가능

> ex) 괄호 위치, 들여쓰기 엉켜 있어도 자동으로 정리해줌.

그래서  CI/CD 파이프라인에서 **`terraform fmt` → `terraform validate` 순서로 자동 검사**를 거치는 게 국룰이라 하므니다.

### 3. plan

- terraform으로 적용할 인프라의 변경 사항에 관한 실행 계획을 생성하는 명령어

```bash
terraform plan
```

![image (15)](/images/2025-05-30-airflow/image (15).png)

사진과 같이  출력되는 결과를 확인해서 어떤 사항이 변경될지, 적용될지, 생성될지 등 사용자가 미리 검토하고 확인하여 이해하는 데 도움을 준다~

기존 (띄워져있는) 인프라에서 어떤 부분이 바꼈는지, 확인할 수 있어서 아주 좋음. (json 파일 형태로 뽑기도 가능)

근데 보통은 **-out=<filename>** 옵션을 주어 실행 (filename으로 plan 결과가 생성되며 이는 바이너리 형태이기 때문에 내용을 확인할 수는 없음)

왜냐하면 위 plan 파일이 생성되면 apply 할 때, 고걸 입력값으로 주어 좀 더 유연하게 apply 할 수 있어서요

```yaml
terraform plan -out=tfplan_poop
terraform apply tfplan_poop
```

### **4. apply**

**terraform apply**는 terraform plan 명령을 기반으로 실행됨.

```bash
terraform apply
```

- 사용자 승인 후 리소스 생성 (`terraform apply -auto-approve` 하면 **yes 없이 바로 실행)**
- 위처럼 플랜 파일 인풋으로 넣어서 실행하는 게 죠음.

### **4.** destroy

- terraform destory 명령은 테라폼 구성에서 관리하는 모든 리소스를 제거하는 명령어

```bash
terraform destroy
```

- 만약 terraform 코드로 구성된 리소스의 일부만 제거하기 위해서는 terraform의 원칙인 선언적 특성에 따라 삭제하려는 항목을 코드에서 제거하고, 다시 terraform apply를 하는 방법이 있음

------

## 3. Terraform의 핵심 개념

### 3.1 Provider / Resource

- **provider**:  테라폼으로 생성할 **인프라의 종류**를 의미(ex. **AWS, Azure, GCP**, ..etc)

```hcl
provider "aws" { 
  region = "us-east-1"
}
#변수로 땡겨쓰는거면
provider "aws" {
  region = var.aws_region
}
```

- **resource**: 실제 생성되는 인프라 단위. **클라우드의 서비스**에 해당됨. ****(ex. **AWS** - *EC2, S3, RDS, Lambda, IAM, VPC* ..etc)

```hcl
resource "aws_instance" "example" {
  ami           = var.ami_id
  instance_type = "t2.micro"

  tags = {
    Name = "MyExampleInstance"
  }
}
```

### **3.2 변수의 종류 - variable vs local**

Terraform에서 변수를 쓰는 방식은 크게 두 가지로 나뉨.

바로 외부에서 주입받는 `variable`이랑, 내부에서 연산하거나 공통으로 쓸 값을 정의하는 `local`임.

- **variables :** 인프라에 사용되는 **변수의 값을 선언하고 할당하는** **데** 사용되는 기능

  → 즉 인프라의 변수는 variables 여기에 저장하면 된다.

  - **인프라에 사용되는 값(예: AMI ID, region, 환경 구분 등)을 외부에서 주입받기 위해 사용하는 변수**임.
  - 예를 들어 `dev`, `prod` 환경마다 인스턴스 타입이나 region이 달라져야 할 때 변수로 처리해두면 깔끔함.
  - 이런 변수들은 보통 `variables.tf` 파일에 선언하고, 실제 값은 `terraform.tfvars` 파일에서 할당함.

```hcl
# variables.tf
variable "ami_id" {
  description = "AMI ID for EC2"
  type        = string
}

# terraform.tfvars 파일에 실제 값 저장
ami_id = "ami-0d5bb3742db8fc264"
```

- ```
  .tfvars
  ```

   파일은 실행 시 자동으로 인식됨

  - `var-file="dev.tfvars"` 형식으로 환경별 값 주입 가능

### local은 뭐임

- 반대로 `local`은 **내부에서 공통적으로 사용할 값이나 계산된 값**을 담는 데 쓰임.

- 예를 들어 태그처럼 반복되는 구조, 혹은 조건문, 접두사같은 건 locals로 처리함.

- 특정 모듈 또는 블록 내에서만 사용되므로 범위가 제한된다. (마치 지역변수같은 느낌)

  - 특정 모듈 또는 블록 내에서 정의되어 있으므로 **외부에서 변경되지 않는다**.

  | **특징**      | **variables**                                                | **locals**                                                   |
  | ------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
  | **참조 범위** | Terraform 구성의 **어디에서나 참조 가능**                    | **해당 모듈에서만 참조 가능**                                |
  | **파일위치**  | [variable.tf](http://variable.tf), tfvars 등 외부에서 입력   | main 안에다 입력                                             |
  | **사용 사례** | 일반적으로 외부에서 제공되는 값을 저장하는 데 사용           | 일반적으로 중간 값을 저장하거나 코드를 더 읽기 쉽게 만드는 데 사용 |
  | **예시**      | - AWS region- VPC ID- Subnet ID,- ...etc                     | - S3 bucket name- EC2 instance tags- Lambda environment variables- ...etc |
  | **📋결론**     | 인프라 서비스의 공통적인 부분의 값은 **variables**를 사용하면 된다. | 서비스의 세부적인 값은 **locals** 사용하면된다.              |

**사용 예시1)** variables : AWS region

```hcl
# variables.tf 
variable "aws_region" {
  type    = string
  default = "us-east-1"
}
```

**사용 예시2)** locals : S3 bucket name

```hcl
# main.tf
locals {
  s3_bucket_name = "my-s3-bucket-${var.aws_region}"
}

주절주절~~
```

### 3.3 State

State는 Terraform이 생성한 인프라의 **현재 상태를 기록하는 파일임.**

즉, **지금 클라우드에 어떤 리소스가 어떤 속성으로 존재하고 있는지를 Terraform이 기억하는 저장소**

![img](https://blog.kakaocdn.net/dn/b5AAOX/btsCUykpx01/JS8p6Kec1Hf8yvck88BZ0K/img.png)

- State는 Terraform이 어떤 리소스를 생성하고 어떤 속성을 가지고 있는지를 추적하는 데 사용됨.

  - 그말은 즉슨 State 파일이 없으면 인프라의 현재 상태를 확인할 수 없게 됨.

- Terraform 명령어를 실행할 때마다 자동으로 state 파일이 업데이트된다.

- 일반적으로 **`.tfstate`**파일 형태로 저장됨.

- State 파일은 일반적으로 다음과 같은 **로컬**이나 **원격 저장소**에 저장함. ( CI/CD에서 s3같이 중간 저장소 느낌으로 사용가능)

  - 로컬 디렉토리
  - S3
  - GCS

- State 파일은 **JSON 형식**으로 작성되며 다음과 같은 정보를 포함한다.

  ```json
  # terraform.tfstate
  {
    "version": 4,
    "terraform_version": "1.11.4",
    "serial": 7,
    "lineage": "fa263314-0b2f-cb96-130d-6289318ac80b",
    "outputs": {},
    "resources": [
      {
        "mode": "managed",
        "type": "aws_instance",
        "name": "ec2_1",
        "provider": "provider[\\"registry.terraform.io/hashicorp/aws\\"]",
        "instances": [
          {
            "schema_version": 1,
            "attributes": {
              "ami": "ami-0d5bb3742db8fc264",
              "arn": "arn:aws:ec2:ap-northeast-2:302263078740:instance/i-05a379ca506a4beb1",
              "associate_public_ip_address": false,
              "availability_zone": "ap-northeast-2d",
              "capacity_reservation_specification": [
                {
                  "capacity_reservation_preference": "open",
                  "capacity_reservation_target": []
                }
              ],
              "cpu_core_count": 1,
              "cpu_options": [
                {
                  "amd_sev_snp": "",
                  "core_count": 1,
                  "threads_per_core": 2
                }
              ],
              "cpu_threads_per_core": 2,
              "credit_specification": [
                {
                  "cpu_credits": "unlimited"
                }
              ],
              "disable_api_stop": false,
              "disable_api_termination": false,
              "ebs_block_device": [],
              "ebs_optimized": true,
              "enable_primary_ipv6": null,
              "enclave_options": [
                {
                  "enabled": false
                }
              ],
  ```

------

------

## 4. 난 그냥 AWS에 프로비저닝 되어있는 거 코드로 빼고 싶어.

- 기존에 AWS 콘솔로 만든 인프라를 `terraform` 코드로 역으로 추출 가능
- 물론 terraform import (terraform 내장기능) 로 빼올 수 있지만 이런점이 불편쓰
  1. 각 리소스를 하나 하나 나열해야 함,
  2. 각 리소스에 맞는 리소스 id을 매칭해야 함.
- [Terraformer](https://github.com/GoogleCloudPlatform/terraformer)는 Google에서 만든 도구

```bash
terraformer import aws \\
  --resources=instance,vpc,subnet \\
  --regions=ap-northeast-2 \\
  --profile=내정보
```

- `-resources`: 가져올 리소스 종류 (예: `instance`, `vpc`, `subnet`, `sg` 등)
- `-regions`: AWS 리전
- `-profile`: AWS CLI에 등록된 IAM 프로파일 이름 (테라포머 install 하고 터미널에 aws iam 이미 연동되어 이씅면 default나 옵션 안써도 ok)

### 특징

- 실제 생성된 리소스를 `.tf` + `.tfstate` 형태로 자동 생성
- 단점: 불필요한 속성도 모두 가져오기 때문에 리팩토링 필요함
  - 이게 ec2 생성할 때 default로 지정해줬던 옵션들도 다 가져 오기 떄문에 매우 코드가 길긴 함..
- 장점: 레거시 환경을 코드 기반으로 전환할 때 매우 유용