# nginx-web-ui

**nginx-web-ui는 nginx를 웹 화면에서 관리하는 도구입니다.** nginx 설정 파일을 직접 편집하지 않고 리버스 프록시(HTTP/HTTPS), TCP/UDP 프록시, 인증서, 캐시와 gzip 압축을 설정할 수 있습니다. 사용자 권한 관리, 변경 이력과 복원, 로그 조회, 여러 노드 관리도 제공합니다.

관리 화면, nginx, 데이터베이스가 **하나의 Docker 이미지**에 들어 있습니다. 별도의 nginx·데이터베이스 설치는 필요하지 않습니다. 필요한 설정 파일은 아래 내용을 복사해 직접 만듭니다.

## 1. 설치 준비와 이미지 선택

- Linux 컨테이너를 실행할 수 있는 Docker가 필요합니다. Linux에서는 Docker Engine과 Compose 플러그인을, Windows/macOS에서는 Linux 컨테이너 모드의 Docker Desktop을 사용할 수 있습니다. [Docker·Compose 설치 안내](https://docs.docker.com/compose/install/)
- 이미지 주소는 `nexus.xelexis.com/nginx-web-ui`입니다.
- 아래 예제는 **1.17.1** 버전을 기준으로 합니다.

다음 명령으로 Docker 연결, Compose 설치, **컨테이너를 실행할 Docker 서버의 아키텍처**를 확인합니다. 원격 Docker context를 사용한다면 명령을 입력하는 PC와 실행 대상이 다를 수 있습니다.

```sh
docker version
docker compose version
docker context show
docker info --format '{{.OSType}}/{{.Architecture}}'
```

| Docker 서버 | 사용할 이미지 |
| --- | --- |
| `linux/x86_64` 또는 `linux/amd64` | `nexus.xelexis.com/nginx-web-ui:1.17.1` |
| `linux/aarch64` 또는 `linux/arm64` | `nexus.xelexis.com/nginx-web-ui:1.17.1-arm64` |

`latest`와 접미사 없는 버전 태그는 **AMD64 전용**입니다. ARM64 환경에서는 반드시 `-arm64` 태그를 선택하세요. 아키텍처가 자동으로 선택되는 공통 태그가 아닙니다.

설치 전에 `docker ps -a`로 기존 컨테이너를 확인하세요. 아래 예제는 새 설치용이며, 기존 `nginx-web-ui` 컨테이너나 `nwu-*` 볼륨이 있다면 먼저 기존 설치인지 확인해야 합니다. 사용 중인 서비스를 임의로 삭제하거나 그 데이터 볼륨을 새 설치에 재사용하지 마세요.

## 2. Docker Compose로 설치

설치 전용 폴더를 만들고 이동합니다. 아래 명령은 Bash와 PowerShell 모두에서 실행할 수 있습니다.

```sh
mkdir nginx-web-ui
cd nginx-web-ui
```

이 폴더에 **`compose.yaml`** 파일을 만들고 다음 내용을 그대로 저장합니다. **ARM64라면 `image`의 태그를 `1.17.1-arm64`로 바꾼 뒤 실행하세요.**

```yaml
services:
  nginx-web-ui:
    image: nexus.xelexis.com/nginx-web-ui:1.17.1
    container_name: nginx-web-ui
    restart: unless-stopped
    ports:
      - "8080:8080"
      - "80:80"
      - "443:443"
    volumes:
      - pg-data:/var/lib/postgresql/data
      - app-data:/var/lib/nginx-web-ui
      - nginx-log:/var/log/nginx
      - app-log:/var/log/nginx-web-ui

volumes:
  pg-data:
    name: nwu-pg
  app-data:
    name: nwu-app
  nginx-log:
    name: nwu-nginx-log
  app-log:
    name: nwu-app-log
```

브라우저 없이 관리자 계정까지 생성할 경우에는 **실행 전에 3절의 `admin.env`와 `env_file` 설정도 추가**하세요. 웹에서 계정을 만들 경우에는 그대로 진행합니다.

같은 폴더에서 설정을 검사하고 이미지를 받아 실행합니다.

```sh
docker compose -f compose.yaml config --quiet
docker compose -f compose.yaml pull
docker compose -f compose.yaml up -d
```

볼륨은 없으면 자동으로 생성됩니다. Windows/macOS에서도 이 예제의 Docker 관리 볼륨을 그대로 사용하세요. 특히 데이터베이스 경로를 호스트 폴더로 바꾸면 파일 권한 문제로 시작하지 못할 수 있습니다.

### 포트와 접속 주소

포트 표기는 **`호스트 포트:컨테이너 포트`**입니다.

| 컨테이너 포트 | 용도 | 기본 접속 |
| --- | --- | --- |
| `8080/tcp` | 관리 화면 | 설치한 컴퓨터에서 `http://localhost:8080` |
| `80/tcp` | HTTP 프록시와 Let's Encrypt HTTP-01 인증 | 등록한 도메인의 HTTP 요청 |
| `443/tcp` | HTTPS 프록시 | 인증서를 연결한 도메인의 HTTPS 요청 |

다른 컴퓨터에서는 `http://<Docker 서버 주소>:8080`으로 접속합니다. `<Docker 서버 주소>`는 실제 설치 대상의 주소로 바꾸세요. 방화벽과 네트워크에서도 필요한 포트가 허용되어야 합니다.

관리 포트가 이미 사용 중이면 `"8080:8080"`을 `"18080:8080"`으로 바꾸고 `http://localhost:18080`을 사용합니다. 설치한 컴퓨터에서만 관리 화면에 접근하려면 `"127.0.0.1:8080:8080"`으로 설정할 수 있습니다. 최초 관리자 생성 전에는 관리 포트에 설치 담당자만 접근할 수 있게 하세요.

내부 웹 서버 `3000/tcp`와 PostgreSQL `5432/tcp`는 별도로 공개하지 않습니다. 컨테이너의 기본 포트와 내부 프로세스 설정은 그대로 사용하세요.

### 영구 보관되는 데이터

| Docker 볼륨 | 컨테이너 경로 | 내용 |
| --- | --- | --- |
| `nwu-pg` | `/var/lib/postgresql/data` | 사용자, 프록시 설정, 감사 로그, 설정 이력 등 데이터베이스 |
| `nwu-app` | `/var/lib/nginx-web-ui` | 인증·암호화 키, DB 접속 비밀값, 인증서 등 앱 데이터 |
| `nwu-nginx-log` | `/var/log/nginx` | nginx 접근·오류 로그 |
| `nwu-app-log` | `/var/log/nginx-web-ui` | 로그인 실패 등 앱 로그 |

**`nwu-pg`와 `nwu-app`은 함께 보존하고 백업해야 합니다.** 앱 볼륨만 지우면 기존 DB에 접속하거나 암호화된 설정을 읽지 못할 수 있습니다. `/etc/nginx`, `conf.d`, `stream.d`에는 별도 볼륨을 덮어씌우지 마세요. nginx 설정은 저장된 논리 설정으로부터 앱이 생성하고 다시 적용합니다.

## 3. 최초 관리자 생성

### 웹 화면에서 생성

1. 컨테이너 부팅 후 `http://localhost:8080/setup`으로 접속합니다. 원격 설치나 포트 변경 시 주소를 맞춥니다.
2. 최초 관리자 이메일과 비밀번호를 입력합니다. 비밀번호는 **8자 이상이며 영문자와 숫자를 포함**해야 합니다.
3. 생성한 계정으로 로그인해 관리 화면이 열리는지 확인합니다.

기본 비밀번호는 없습니다. 관리자 비밀번호가 자동 생성되거나 로그에 출력되지도 않습니다. 사용자가 생성된 뒤에는 최초 설정 화면을 다시 사용할 수 없습니다.

### 에이전트·자동화로 관리자까지 생성

브라우저의 최초 설정을 생략하려면 **첫 실행 전에** 운영자가 정한 이메일과 비밀번호를 전달합니다. `compose.yaml`과 같은 폴더에 `admin.env`를 만들고 다음 두 값을 실제 값으로 바꾸세요.

```dotenv
ADMIN_EMAIL=admin@example.com
ADMIN_PASSWORD='REPLACE_WITH_YOUR_UNIQUE_PASSWORD_123'
```

예제 비밀번호를 그대로 사용하지 마세요. Compose의 환경 파일에서 비밀번호를 작은따옴표로 감싸면 `$` 같은 문자를 그대로 전달할 수 있습니다. 파일에는 설치 담당자만 접근하게 하고 저장소에 커밋하지 않습니다. [Compose 환경 파일 문법](https://docs.docker.com/compose/how-tos/environment-variables/variable-interpolation/#env-file-syntax)

`compose.yaml`의 `nginx-web-ui` 서비스 안에 다음 설정을 추가합니다. `env_file`의 들여쓰기는 `image`, `ports`, `volumes`와 같은 수준입니다.

```yaml
    env_file:
      - ./admin.env
```

이후 위의 `pull`·`up -d` 명령을 실행합니다. 빈 DB이면 계정이 생성되고 `/setup` 대신 로그인 화면을 사용합니다. 기존 사용자가 있으면 이 환경변수로 비밀번호가 변경되지 않습니다.

기본 설치에는 이외의 환경변수가 필요하지 않습니다. `AUTH_SECRET`과 내장 DB 접속 비밀값은 최초 부팅 시 생성되어 앱 볼륨에 보관됩니다. `DATABASE_URL`을 별도로 지정하거나 외부 DB를 준비하지 마세요.

## 4. 설치 성공 확인

```sh
docker compose -f compose.yaml ps
docker compose -f compose.yaml logs --tail=100 nginx-web-ui
docker exec nginx-web-ui nginx -t
docker exec nginx-web-ui pg_isready -h 127.0.0.1 -p 5432
```

`nginx -t`는 설정 검사 성공, `pg_isready`는 `accepting connections`가 나와야 합니다. 컨테이너가 `running`이라는 것만으로 웹 서비스가 준비됐다고 판단하지 마세요. 초기화 중이면 잠시 기다린 뒤 HTTP까지 확인합니다.

Bash에서는 다음 명령을 사용합니다.

```sh
curl --fail --silent --show-error --location --output /dev/null --write-out '%{http_code}\n' http://localhost:8080/setup
```

PowerShell에서는 다음 명령을 사용합니다.

```powershell
(Invoke-WebRequest -Uri 'http://localhost:8080/setup').StatusCode
```

정상적으로 부팅된 새 설치는 최초 설정 페이지에서 **HTTP 200**을 반환합니다. 관리자가 이미 생성된 경우에는 로그인 화면으로 이동합니다. 최종적으로 실제 계정으로 로그인되는지 확인하세요. 원격 Docker 서버나 다른 호스트 포트를 사용했다면 확인 URL도 변경해야 합니다.

## 5. 프록시 사용 시작

- 관리 화면에서 업스트림(대상 서비스)을 등록하고 리버스 프록시의 도메인·포트·대상을 설정한 뒤 적용합니다. 캐시와 gzip 압축은 서버 단위로 설정하고 경로별로 켜거나 끌 수 있습니다.
- 프록시 대상으로 입력하는 `localhost`·`127.0.0.1`은 **nginx-web-ui 컨테이너 자신**입니다. 다른 컨테이너는 같은 Docker 네트워크에 연결한 뒤 서비스 이름과 내부 포트로 지정합니다. 외부 서비스는 컨테이너에서 접근할 수 있는 주소와 포트를 사용합니다. [Docker 네트워크 안내](https://docs.docker.com/engine/network/)
- 관리 화면에서 포트를 추가해도 Docker 포트가 자동으로 공개되지는 않습니다. 예를 들어 컨테이너에서 TCP `15432`를 사용하는 stream은 `ports`에 `"15432:15432/tcp"`를, UDP `15353`은 `"15353:15353/udp"`를 추가하고 `docker compose -f compose.yaml up -d`를 다시 실행합니다. TCP/UDP 구분도 맞춰야 합니다.
- Let's Encrypt HTTP-01 인증서를 발급하려면 도메인의 DNS A/AAAA가 설치 대상으로 연결되고 인터넷에서 **80번 포트**에 접근할 수 있어야 합니다. 발급한 인증서를 프록시에 연결하고 **443번 포트**로 HTTPS를 제공합니다. HTTP-01을 사용할 수 없는 환경은 인증서 화면에서 DNS-01 방식을 선택할 수 있습니다.
- 프록시 도메인을 등록하기 전 `http://localhost:80`의 **404는 정상**입니다. 기본 관리 화면 주소는 `:8080`입니다. 인증서를 연결하지 않은 도메인의 443 접속에는 기본 인증서 경고가 나타날 수 있습니다.

## 6. 업데이트와 중지

업데이트 전에 [버전별 업그레이드 경로](UPGRADE_PATH.md)를 확인하세요. 필수 경유 버전과 현재 버전에서 챙길 조건을 안내합니다.

업데이트 전에는 변경 내역을 확인하고 DB·앱 볼륨을 함께 백업합니다. `compose.yaml`의 `image`를 원하는 버전으로 바꾼 뒤 **기존 설치 폴더에서** 실행하세요. ARM64는 새 버전에도 `-arm64` 접미사를 붙입니다.

1.16.x에서 1.17.x로 올리며 여러 노드를 함께 관리하는 경우, **게이트웨이와 연결된 노드를 모두 1.17.x로 맞춰야 합니다.** 서로 다른 두 버전 사이에서는 관리 연결이 차단됩니다. 사용자 비밀번호와 인증서가 보존되도록 DB·앱 데이터를 함께 백업하고, 기존 실행 설정과 이미지도 되돌릴 수 있게 보관하세요.

```sh
docker compose -f compose.yaml pull
docker compose -f compose.yaml up -d
```

Compose는 이미지가 바뀌면 컨테이너를 재생성하며 연결된 볼륨을 유지합니다. 완료 후 위의 설치 성공 확인을 다시 수행하세요. [Compose 업데이트 동작](https://docs.docker.com/reference/cli/docker/compose/up/)

일시 중지와 다시 시작:

```sh
docker compose -f compose.yaml stop
docker compose -f compose.yaml start
```

컨테이너를 제거하되 데이터를 유지하려면 `docker compose -f compose.yaml down`을 사용합니다. **`down -v`는 데이터 볼륨까지 삭제하므로 일반 중지·업데이트에 사용하지 마세요.**

## 7. 설치 문제 해결

| 증상 | 확인할 내용 |
| --- | --- |
| Docker에 연결할 수 없음 | Docker 서비스 실행 여부, 현재 context, Docker 명령 실행 권한 확인 |
| `port is already allocated` | 호스트에서 해당 포트를 쓰는 서비스 확인. 관리 포트는 호스트 쪽 숫자를 변경. 기존 서비스를 임의로 중지하지 않음 |
| 아키텍처 불일치·`exec format error` | `docker info` 결과와 이미지 태그 비교. ARM64는 `-arm64` 사용 |
| 이미지 다운로드 실패 | 이미지 주소와 버전 태그, 네트워크·DNS·레지스트리 HTTPS 연결 확인 |
| 컨테이너는 실행 중인데 관리 화면에 접속 불가 | `docker logs --tail=100 nginx-web-ui`, `nginx -t`, `pg_isready` 결과와 호스트 포트·방화벽 확인 |
| 새 설치인데 `/setup`이 나오지 않음 | 기존 DB 볼륨 또는 `ADMIN_PASSWORD`로 계정이 이미 생성됐는지 확인 |
| 프록시가 502를 반환 | 업스트림 주소·포트와 Docker 네트워크 확인. `localhost`가 대상 서버를 가리키는지 재확인 |
| DB 비밀번호나 암호화 키 관련 오류 | `nwu-pg`와 `nwu-app`이 같은 설치의 볼륨인지 확인. 한쪽만 삭제·교체하지 말고 함께 보존한 백업 확인 |
