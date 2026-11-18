# indico-ansible

CentOS 7 서버 한 대에 [Indico](https://getindico.io)를 설치하는 Ansible 플레이북입니다.

Indico는 CERN이 만든 오픈소스 행사·학회 관리 시스템입니다. [공식 설치 문서](https://docs.getindico.io/en/stable/installation/production/centos/)의 수동 설치 절차를 그대로 role로 옮겼습니다.

> CentOS 7은 2024년 6월에 지원이 끝났습니다. 이 플레이북은 EL7 기준으로 만들었고, 다른 배포판에서는 테스트하지 않았습니다.

## 설치되는 것

| 구성 요소 | 버전·방식 |
|---|---|
| PostgreSQL | 13 (PGDG 저장소) |
| Redis, nginx | EPEL 패키지 |
| Python | pyenv로 3.9.16 빌드, `/opt/indico/.venv` 가상환경 |
| Indico, uWSGI | pip 설치, systemd 서비스로 등록 (`indico-uwsgi`, `indico-celery`) |
| TLS | 자체 서명 인증서 |
| SELinux | 끄지 않고 Indico용 정책 모듈 적용 |
| firewalld | 80, 443 포트 개방 |

## 실행 방법

1. `inventory/hosts.ini`에 대상 서버 주소를 적습니다.

   ```ini
   [indico]
   worker1 ansible_host=192.168.56.102 ansible_user=root
   ```

2. 관리자 이메일 같은 값은 `group_vars/all.yml`에서 바꿉니다. 접속 주소(`indico_hostname`)는 기본으로 `ansible_host`를 씁니다.

3. 필요한 컬렉션을 받고 실행합니다.

   ```bash
   ansible-galaxy collection install -r requirements.yml
   ansible-playbook site.yml
   ```

설치가 끝나면 `https://<서버 주소>`로 접속해 첫 관리자 계정을 만듭니다.

## 구조

```
.
├── ansible.cfg
├── inventory/hosts.ini
├── group_vars/all.yml        # 주소, 경로, 버전 변수
├── requirements.yml
├── site.yml
└── roles
    ├── repo          # EPEL, SCL, PGDG 저장소. 기본 저장소의 postgresql 제외
    ├── packages      # PostgreSQL 13, Redis, nginx, 빌드용 라이브러리
    ├── database      # initdb, indico 사용자와 DB, unaccent·pg_trgm 확장
    ├── services      # uWSGI 설정, systemd 유닛
    ├── nginx_ssl     # nginx 사이트 설정, DH 파라미터, 자체 서명 인증서
    ├── selinux       # Indico SELinux 정책 모듈
    ├── python_env    # indico 사용자, pyenv, venv, pip 설치, setup wizard
    ├── indico_app    # 권한 정리, 정적 파일 설정, db prepare, 서비스 시작
    └── firewall      # 80, 443 개방
```

설정 파일은 role의 `templates/`, `files/`에 두고 `template`, `copy` 모듈로 배포합니다. initdb, DB 생성, pyenv 빌드, setup wizard, `indico db prepare`는 이미 되어 있으면 건너뛰어서, 같은 서버에 다시 실행해도 됩니다.

## 설계 메모

- **PostgreSQL 패키지 분리**: CentOS 기본 저장소의 `postgresql` 패키지가 PGDG 13 패키지와 섞이지 않도록 `CentOS-Base.repo`의 `[base]`, `[updates]`에 `exclude=postgresql*`를 넣습니다.
- **SELinux 유지**: SELinux를 끄지 않고, nginx가 uWSGI 소켓과 로그 디렉터리에 접근할 수 있게 정책 모듈(`roles/selinux/files/indico.cil`)을 올립니다.
- **설치 마법사 자동 응답**: `indico setup wizard`는 대화형이라 `expect` 모듈로 답을 넣습니다. 프롬프트 색 코드가 매칭을 방해하지 않도록 `PROMPT_TOOLKIT_COLOR_DEPTH=DEPTH_1_BIT`로 색을 끄고 `sed`로 남은 코드를 지웁니다.

## 한계

- setup wizard 응답은 특정 Indico 버전의 질문 순서에 맞춰져 있어서, 버전이 바뀌면 다시 맞춰야 합니다.
- 서버 한 대 구성만 다룹니다. DB 분리나 이중화는 없습니다.
- 인증서는 자체 서명이라 실제 서비스에는 따로 발급받은 인증서로 바꿔야 합니다.
