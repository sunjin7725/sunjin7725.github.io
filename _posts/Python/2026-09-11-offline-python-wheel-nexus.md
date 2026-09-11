---
title: 폐쇄망에서 Python 패키지 사용하기 — wheel 다운로드부터 Nexus 등록까지
description: 대상 환경에 맞는 wheel과 의존성을 외부망에서 준비하고 폐쇄망 Nexus PyPI 저장소에 등록해 설치하는 과정
categories: [Python]
tags: [python, pip, wheel, nexus, offline]
comments: true
toc: true
---

## 패키지 파일 하나만 가져오면 될까?

폐쇄망에서 사용할 Python 패키지를 준비하다 보면, 필요한 `.whl` 파일만 내려받으면 끝날 것 같지만 실제로는 의존성이나 플랫폼 호환성에서 막히는 경우가 있다. 최근 impyla를 준비할 때도 wheel만 받도록 설정하자 소스 배포본만 있는 의존성이 제외되면서 다운로드가 실패했다.

이번 글에서는 특정 PC 운영체제에 한정하지 않고, **외부망에서 패키지와 의존성을 준비해 폐쇄망 Nexus에서 설치하는 과정**을 정리한다. 다운로드 PC보다 중요한 기준은 패키지가 실제로 실행될 서버의 환경이다.

```text
외부망 준비 환경
  패키지·의존성 다운로드 → 필요하면 wheel 빌드 → 설치 검증
                              ↓
                     파일을 내부망으로 반입
                              ↓
폐쇄망 Nexus PyPI hosted에 업로드
                              ↓
내부 서버에서 Nexus를 통해 pip install
```

Nexus는 패키지 파일을 저장하고 배포하는 역할을 한다. Linux용 wheel을 Windows용으로 변환하거나 누락된 의존성을 자동으로 만들어주지는 않는다.

> 아래 주소와 저장소 이름은 예시다. 실제 Nexus에 업로드·설치를 완료한 기록이 아니라, 공식 문서를 바탕으로 정리한 절차이며 환경에 맞춰 검증해야 한다.
{: .prompt-info }

## 1. 설치 대상의 Python과 플랫폼 확인하기

먼저 **폐쇄망의 설치 대상 서버**에서 실행한다. 아래 명령의 `python`은 사용할 인터프리터에 맞게 `python3` 또는 `py -3.10` 등으로 바꾼다.

```bash
python --version
python -m pip --version
python -c "import platform; print(platform.system(), platform.machine(), platform.python_implementation())"
python -m pip debug --verbose
```

`pip debug --verbose`의 Compatible tags에서 해당 인터프리터가 설치할 수 있는 wheel 태그를 확인할 수 있다.

| 확인 항목 | 예시 |
|---|---|
| Python 구현·버전 | CPython 3.10 |
| 운영체제 | Linux 또는 Windows |
| CPU 아키텍처 | x86_64 또는 ARM64 |
| Linux C 라이브러리 | glibc 또는 musl |
| 네이티브 실행 의존성 | ODBC 라이브러리, GPU 드라이버 등 |

wheel 파일명은 다음과 같은 정보를 담는다.

```text
example-1.0.0-cp310-cp310-manylinux_2_17_x86_64.whl
              │     │              │
            Python  ABI        플랫폼 조건
```

`py3-none-any`는 플랫폼에 종속되지 않는 Python 3 wheel을 나타낸다. 그래도 패키지의 Python 버전 요구사항이나 실행 시 외부 프로그램 필요 여부까지 사라지는 것은 아니다. `manylinux`와 `musllinux`도 구분해야 한다. [Python 패키징 호환성 태그 문서](https://packaging.python.org/en/latest/specifications/platform-compatibility-tags/)

## 2. 반입할 패키지 목록 정하기

`requirements.txt`를 만들고 필요한 버전을 지정한다. 다음은 설명을 위한 고정 버전 예시이며, 최신 버전 추천 목록은 아니다.

```text
requests==2.32.5
```

실제 작업에서는 프로젝트에서 검증할 패키지와 버전을 적는다. 환경별로 조건이 달라질 수 있으므로 Linux와 Windows의 반입 폴더는 분리하는 편이 관리하기 쉽다.

```text
package-bundle/
├── requirements.txt
├── requirements.lock.txt
└── wheelhouse/
    ├── 패키지.whl
    └── 의존성.whl
```

상위 패키지의 버전만 고정하면 하위 의존성은 다운로드 시점에 따라 달라질 수 있다. 뒤에서 깨끗한 환경에 설치한 뒤 의존성 버전 목록도 보관한다.

## 3. 외부망에서 의존성까지 다운로드하기

### 실제 다운로드에 사용한 명령

impyla를 준비할 때 실제로 다운로드가 성공한 명령은 다음과 같다. 대상은 **Windows 64bit / Python 3.7**이며, 현재 폴더의 배포 파일도 검색 후보에 포함했다.

```bash
python -m pip download \
  impyla \
  --only-binary=:all: \
  --python-version=3.7 \
  --platform=win_amd64 \
  --find-links=./
```

`--find-links=./`는 현재 폴더의 wheel 등을 **검색 후보로 추가**한다. 저장 위치를 정하는 옵션은 아니다. 이 명령은 `--dest`를 생략했으므로 결과도 기본값인 현재 폴더에 저장된다. `--no-index`가 없으므로 설정된 패키지 인덱스도 함께 조회한다.

`impyla` 버전을 고정하지 않았으므로 선택되는 버전은 인덱스와 로컬 파일, 호환 조건에 따라 달라질 수 있다. Python 3.7은 이 사례의 대상 버전이며 새 환경에 권장하는 버전이라는 의미는 아니다. 다운로드 성공과 대상 서버의 설치·실행 성공은 구분해서 확인한다.

위 줄바꿈은 Bash·zsh 기준이다. PowerShell에서는 다음 한 줄 명령을 사용할 수 있다.

```powershell
python -m pip download impyla --only-binary=:all: --python-version=3.7 --platform=win_amd64 --find-links=./
```

아래는 이 사례와 구분되는 일반 예시다. `--implementation`과 `--abi`는 대상 구현과 ABI까지 명시하는 옵션으로, 위 성공 명령에는 지정하지 않았다.

### 대상과 같은 환경에서 준비하는 경우

외부망에 대상과 같은 OS·아키텍처·Python 환경을 마련할 수 있다면 우선 이 방법을 사용한다.

```bash
python -m pip download --only-binary=:all: --dest wheelhouse -r requirements.txt
```

`pip download`는 설치 대신 배포 파일을 수집하며 의존성도 함께 해석한다. `--only-binary=:all:`은 소스 배포본을 제외하고 wheel만 허용한다. 따라서 필요한 의존성에 wheel이 없다면 실패하는 것이 정상이다. [pip download 문서](https://pip.pypa.io/en/stable/cli/pip_download/)

### 다른 환경에서 다운로드하는 경우

예를 들어 준비 PC가 Windows나 macOS이고 대상이 **Linux x86_64 / CPython 3.10**이라면 대상 조건을 명시할 수 있다.

```bash
python -m pip download --only-binary=:all: --python-version 3.10 --implementation cp --abi cp310 --platform manylinux2014_x86_64 --platform manylinux_2_17_x86_64 --dest wheelhouse -r requirements.txt
```

두 플랫폼 태그는 glibc 2.17 기준의 해당 wheel 표기들을 대상으로 한 예시다. 대상에서 지원하는 모든 태그를 나열한 것은 아니다. 필요한 wheel이 더 높은 glibc 기준으로만 제공된다면 대상의 Compatible tags를 확인해 지원되는 태그를 추가한다.

Windows 64bit / CPython 3.10이 대상이라면 다음과 같이 지정한다.

```bash
python -m pip download --only-binary=:all: --python-version 3.10 --implementation cp --abi cp310 --platform win_amd64 --dest wheelhouse -r requirements.txt
```

명령은 Bash·PowerShell에서 줄 연결 문법을 바꾸지 않아도 되도록 한 줄로 적었다. 다운로드만 다른 플랫폼 대상으로 수행하는 것이며, 현재 PC에서 그 플랫폼의 패키지를 실행할 수 있게 되는 것은 아니다.

교차 다운로드 옵션을 지정해도 대상 환경을 완전히 재현한 검증은 아니다. 특히 OS·Python에 따라 조건부 의존성이 달라지는 패키지는 대상과 같은 환경에서 의존성을 다시 확인한다.

## 4. wheel이 없는 의존성 처리하기

다음 오류가 나왔다고 해서 바로 버전 충돌이라고 단정하지 않는다.

```text
No matching distribution found
ResolutionImpossible
```

먼저 해당 버전의 Python 요구사항, 플랫폼 태그, wheel 제공 여부를 살펴본다. `--only-binary` 때문에 후보가 제외된 것인지도 확인한다.

예를 들어 `pure-sasl==0.6.2`는 확인 시점에 PyPI에 소스 배포본만 제공된다. 이런 의존성은 별도로 wheel을 만든 뒤 다운로드 과정에 포함할 수 있다. [pure-sasl 0.6.2 배포 파일](https://pypi.org/project/pure-sasl/0.6.2/#files)

```bash
python -m pip wheel --no-deps --wheel-dir built-wheels "pure-sasl==0.6.2"
```

이 명령은 **인터넷에 연결된 빌드 환경**에서 실행한다. `--no-deps`는 패키지의 실행 의존성을 함께 빌드하지 않는다는 뜻이며, 빌드에 필요한 도구 다운로드까지 차단하는 옵션은 아니다. [pip wheel 문서](https://pip.pypa.io/en/stable/cli/pip_wheel/)

생성된 wheel의 태그를 확인한 후, 원래 프로젝트의 `requirements.txt`로 다시 다운로드한다.

```bash
python -m pip download --only-binary=:all: --find-links built-wheels --dest wheelhouse -r requirements.txt
```

다른 플랫폼을 대상으로 받던 경우에는 앞 단계의 `--python-version`, `--implementation`, `--abi`, `--platform` 옵션도 그대로 추가한다. 의존성 해결이 성공하면 필요한 로컬 wheel과 나머지 의존성이 `wheelhouse`에 수집된다.

**네이티브 확장이 있는 패키지는 대상에 호환되는 환경에서 빌드해야 한다.** macOS에서 만든 바이너리 wheel의 파일명만 바꿔 Linux에 가져갈 수는 없다. Linux 컨테이너로 빌드하더라도 아키텍처·glibc·외부 라이브러리 조건까지 확인한다.

## 5. Nexus 등록 전에 오프라인 설치 검증하기

외부망에 마련한 대상과 같은 테스트 환경에서 새 가상환경을 만든다.

```bash
python -m venv verify-env
```

가상환경을 활성화한다.

Linux·macOS의 Bash 계열 셸:

```bash
source verify-env/bin/activate
```

Windows PowerShell:

```powershell
.\verify-env\Scripts\Activate.ps1
```

이후에는 공통 명령을 사용한다.

```bash
python -m pip install --no-index --find-links wheelhouse --only-binary=:all: -r requirements.txt
python -m pip check
python -c "import requests; print(requests.__version__)"
python -m pip freeze > requirements.lock.txt
```

`--no-index`를 사용하면 패키지 인덱스에 접근하지 않고 로컬 폴더에서 설치한다. 위처럼 일반 패키지명과 버전으로 작성한 requirements를 사용한다. 직접 URL이나 VCS 주소가 들어 있다면 별도로 정리해야 한다. [pip install 문서](https://pip.pypa.io/en/stable/cli/pip_install/)

import 예제는 실제 패키지에 맞게 바꾼다. `pip check`는 선언된 Python 의존성을 검사하므로, DB 연결이나 GPU 연산 같은 대표 기능도 별도로 실행해야 한다. 시스템 라이브러리와 드라이버는 wheel 수집만으로 해결되지 않을 수 있다.

`requirements.lock.txt`는 이 테스트 환경의 설치 버전 목록이다. 모든 OS에 통용되는 잠금 파일이나 해시 검증 파일은 아니다. 반입 파일과 함께 대상 환경 정보도 보관한다.

## 6. 폐쇄망 Nexus의 PyPI 저장소에 등록하기

### hosted, proxy, group 구분하기

| 유형 | 역할 |
|---|---|
| PyPI hosted | 반입한 패키지를 직접 업로드하고 보관 |
| PyPI proxy | 외부 PyPI 등의 패키지를 받아 캐시 |
| PyPI group | 여러 저장소를 하나의 조회 주소로 묶음 |

직접 반입한 파일을 넣는 곳은 **PyPI hosted**다. 외부 접속이 불가능한 proxy는 아직 캐시되지 않은 패키지를 가져올 수 없다. group을 사용한다면 반입한 hosted가 구성원에 포함되어야 한다. [Sonatype PyPI 저장소 문서](https://help.sonatype.com/en/pypi-repositories.html)

Nexus 관리자에게 `pypi (hosted)` 형식의 저장소를 준비하도록 요청하거나, 권한이 있다면 생성한다. 여기서는 저장소 이름을 `pypi-internal`로 가정한다.

### Twine으로 업로드하기

Twine은 **Nexus에 접속할 수 있는 내부망 PC**에서 실행한다. 내부망에 Twine이 없다면 업로드 PC의 Python·OS를 기준으로 Twine과 의존성도 외부망에서 따로 수집해 반입한다. 애플리케이션용 wheel과 폴더를 분리하면 업로드 도구 의존성이 섞이지 않는다.

외부망의 업로드 PC와 호환되는 준비 환경:

```bash
python -m pip download --only-binary=:all: --dest upload-tools twine
```

반입 후 내부망 업로드 PC의 별도 가상환경:

```bash
python -m pip install --no-index --find-links upload-tools twine
```

wheel 업로드:

```bash
python -m twine upload --repository-url https://nexus.example.internal/repository/pypi-internal/ wheelhouse/*.whl
```

인증에는 저장소 업로드 권한이 있는 계정을 사용한다. 셸 기록에 비밀번호를 넣는 대신 대화형 입력이나 조직에서 정한 자격증명 방식을 사용한다. Twine의 업로드 주소에는 `/simple/`을 붙이지 않는다. [Sonatype PyPI CLI 사용 문서](https://help.sonatype.com/en/pypi-cli-usage.html)

이미 같은 파일이 존재하면 재배포 정책에 따라 거부될 수 있다. 이 경우 기존 파일과 버전을 확인하고, 오류를 숨긴 채 전체 성공으로 처리하지 않는다.

## 7. 내부 서버에서 Nexus를 통해 설치하기

설치 대상 서버의 새 가상환경에서 실행한다.

```bash
python -m pip install --index-url https://nexus.example.internal/repository/pypi-internal/simple/ --only-binary=:all: -r requirements.lock.txt
python -m pip check
```

설치 주소는 PyPI 조회 API인 **`/simple/`로 끝난다.** group을 통해 배포한다면 저장소 이름을 실제 group 이름으로 변경한다.

기존 pip 설정에 외부 인덱스가 추가되어 있지 않은지도 확인한다. 이 글에서는 외부 PyPI를 `--extra-index-url`로 함께 사용하지 않고, 내부 인덱스를 명시한다. Nexus가 읽기 인증을 요구하면 내부 계정도 설정해야 한다.

사내 인증서를 사용하는 환경이라면 신뢰할 CA를 설정한다. PEM 인증서 묶음을 전달받은 경우에는 다음처럼 지정할 수 있다.

```bash
python -m pip install --cert company-ca.pem --index-url https://nexus.example.internal/repository/pypi-internal/simple/ --only-binary=:all: -r requirements.lock.txt
```

업로드 시에도 같은 인증서가 필요하다면 Twine 명령에 `--cert company-ca.pem`을 추가한다. 인증서 검증을 끄는 방식보다 실제 인증서 체인을 맞추는 편이 적절하다.

## 자주 막히는 부분

| 증상 | 먼저 확인할 항목 |
|---|---|
| 다운로드할 버전이 없다고 나옴 | Python 요구사항, wheel 존재 여부, 지정한 플랫폼 |
| 다운로드는 성공했는데 설치 실패 | 조건부 의존성 누락, 실제 대상의 호환 태그 |
| wheel을 지원하지 않는다고 나옴 | Python·ABI·OS·CPU 아키텍처 |
| Nexus 업로드 401 / 403 | 인증 정보와 저장소 업로드 권한 |
| Nexus에서 패키지를 찾지 못함 | `/simple/` 주소, hosted 등록 상태, group 구성 |
| 설치 후 import 시 공유 라이브러리 오류 | ODBC 등 시스템 라이브러리와 런타임 의존성 |
| 설치 후 실제 기능이 실패 | 서버 연결, 인증, 드라이버와 대표 실행 경로 |

폐쇄망 패키지 준비의 기준은 파일을 몇 개 내려받았는지가 아니라 **대상과 같은 깨끗한 환경에서, 반입할 파일만으로 설치와 실행이 가능한지**다. 이 검증을 마친 패키지를 Nexus에 등록하면 내부 서버에서도 같은 버전을 반복해서 설치하고 관리하기 쉬워진다.
