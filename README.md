# Indico 배포 자동화 (EL7 기반)

> Ansible을 사용한 CentOS 7 (EL7) 환경 Indico 자동 배포 프로젝트

---

## 개요 (Overview)

이 프로젝트는 CentOS 7 (EL7) 환경에서 **Indico**를 자동으로 배포하기 위해 Ansible을 활용하여 구성한 인프라 자동화 플레이북입니다.


### Indico
Indico는 웹 기반 행사 관리 플랫폼으로, 소규모 강연부터 대형 국제 학회까지 다양한 규모의 이벤트를 효율적으로 운영할 수 있도록 지원합니다. 본 프로젝트는 CERN에서 개발된 오픈소스 소프트웨어이며, 전 세계 150개 이상의 기관에서 활용되고 있습니다.

### Indico Installation Guide 
https://docs.getindico.io/en/stable/#

### 프로젝트 목표

- 수동 설치 과정을 자동화
- 재현 가능한 배포 환경 구축
- 역할(Role) 기반 구조로 유지보수성 향상

---

## 아키텍처 구성

본 배포는 다음 구성 요소를 포함합니다:

- PostgreSQL 13
- Redis
- Nginx
- uWSGI
- Python Virtual Environment
- systemd 서비스 등록
- SELinux 정책 적용
- firewalld 포트 설정


### 배포 흐름

1. Repository 및 패키지 설치
2. PostgreSQL 초기화 및 데이터베이스 생성
3. Python 가상환경 구성
4. Indico 설치
5. systemd 서비스 등록
6. SELinux 정책 적용
7. 방화벽 포트 오픈

---

## 프로젝트 구조


각 역할(Role)은 책임 단위로 분리하여 가독성과 유지보수성을 높이도록 설계하였습니다.

역할 별로 tasks/main.yml에 playbook을 작성했습니다.

```
indico-ansible
├── LICENSE
├── ansible.cfg
├── hosts
├── site.yml
└── roles
    ├── database
    │   └── tasks
    │       └── main.yml
    │
    ├── firewall
    │   └── tasks
    │       └── main.yml
    │
    ├── indico_app
    │   └── tasks
    │       └── main.yml
    │
    ├── nginx_ssl
    │   └── tasks
    │       └── main.yml
    │
    ├── packages
    │   └── tasks
    │       └── main.yml
    │
    ├── python_env
    │   └── tasks
    │       └── main.yml
    │
    ├── repo
    │   └── tasks
    │       └── main.yml
    │
    ├── selinux
    │   └── tasks
    │       └── main.yml
    │
    └── services
        └── tasks
            └── main.yml
```

---

## 실행 방법

```
ansible-playbook -i inventory site.yml
```


## 설계 고려 사항 (Design Considerations)

- **역할(Role) 기반 구조 분리**
  - 기능 단위로 역할을 분리하여 가독성과 유지보수성을 개선
  - 책임 단위 분리를 통해 확장 가능하도록 설계

- **systemd 기반 서비스 관리**
  - uWSGI 및 Celery를 systemd로 통합 관리
  - Unit 파일 생성 후 `daemon-reload` 명시적 수행

- **PostgreSQL 버전 충돌 방지**
  - EL7 기본 저장소의 PostgreSQL 패키지와 충돌 방지를 위해  
    `exclude=postgresql*` 설정 적용

- **SELinux 비활성화 대신 정책 적용**
  - 보안을 유지하기 위해 SELinux를 끄지 않고 Custom Policy 모듈 적용

- **명확한 배포 순서 구성**
  - DB 초기화 → 서비스 등록 → 정책 적용 → 방화벽 오픈 순서로 구성
  - 서비스 의존성과 실행 순서를 고려하여 설계

---

## 한계점 (Limitations)

- 일부 `shell` 기반 작업은 완전한 멱등성(idempotency)을 보장하지 않음
- 특정 설정 값이 하드코딩되어 있어 유연성이 제한적
- EL7 환경에 최적화되어 있으며, 다른 배포판에 대한 일반화 부족
- 멀티 노드 또는 고가용성(HA) 환경은 고려되지 않음

---
