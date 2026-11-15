# releases.soneyko.ai

Единый источник готовых релизов программного обеспечения Soneyko. Исходный код находится в репозитории соответствующего проекта; здесь описан обязательный контракт сборки, хранения, публикации и обновления.

Публичные файлы раздаются по HTTPS через `https://releases.soneyko.ai`. Хранилище артефактов — Cloudflare R2. S3 endpoint используется только для публикации и административной проверки, публичный custom domain — для скачивания. Контейнерные образы в этот контракт не входят.

> Этот README — спецификация и пример настройки. Сам по себе он не создаёт bucket, DNS, secrets, GitHub Actions или готовые сборки. Скрипт сборки каждого приложения реализуется в его исходном репозитории.

## 1. Обязательные правила релиза

- Каждый проект имеет постоянный уникальный slug: например, `node-agent`. Разрешены строчные латинские буквы, цифры и одиночные дефисы; начало и конец — буква или цифра. Slug не меняется при переименовании GitHub-репозитория.
- **Название GitHub Release строго `Build 1`, `Build 2`, `Build 3` и далее.** Тег: `build-1`, `build-2`, `build-3`. В пути хранилища используется тот же тег.
- Нумерация отдельная для каждого проекта, не для всей организации. Дополнительную semver-версию можно указать в описании и manifest, но она не заменяет `Build N`.
- Один постоянный workflow `release.yml` владеет нумерацией проекта. В приведённом примере `N = github.run_number`. Неудачная попытка расходует номер: пропуски допустимы, повторное использование номера запрещено.
- Повторный запуск jobs (`Re-run`) не является новым Build. Для новой сборки запускается новый workflow run. Нельзя удалять/пересоздавать workflow или менять схему счётчика без сохранения монотонной нумерации.
- Build привязан к точному полному Git commit SHA. Сборки разных исходных commits под одним номером недопустимы.
- Релиз публикуется только после успешной сборки и проверки **всей обязательной матрицы**. Отсутствующая архитектура или версия ОС — ошибка релиза, а не молчаливый пропуск.
- Все файлы в `builds/build-N/` неизменяемы. Нельзя перезаписывать старый архив, manifest или подпись. Исправление выпускается новым Build.
- Канал `stable` обновляется только после загрузки, обратного чтения и проверки всех файлов и создания GitHub Release.
- Секреты, `.env`, приватные ключи, конфиги машин и пользовательские данные не включаются в артефакты.

## 2. Обязательные дистрибутивы и архитектуры

Архитектуры перечислены **для каждой версии отдельно**. Нельзя строить декартово произведение всех версий и всех архитектур.

- **AlmaLinux**
  - 10: `amd64`, `arm64`.
  - 9: `amd64`, `arm64`.
  - 8: `amd64`, `arm64`.
- **Amazon Linux**
  - 2023: `amd64`, `arm64`.
  - 2: `amd64`, `arm64`.
- **CentOS / CentOS Stream**
  - 10: `amd64`, `arm64`.
  - 9: `amd64`, `arm64`.
  - 8: `amd64`, `arm64`.
  - 7: `amd64`.
- **Debian**
  - 13 Trixie: `amd64`, `arm64`.
  - 12 Bookworm: `amd64`, `arm64`.
  - 11 Bullseye: `amd64`.
  - 10 Buster: `amd64`, `i386`.
- **openSUSE Leap**
  - 16: `amd64`, `arm64`.
  - 15: `amd64`, `arm64`.
- **Oracle Linux**
  - 10: `amd64`, `arm64`.
  - 9: `amd64`, `arm64`.
  - 8: `amd64`, `arm64`.
  - 7: `amd64`.
- **Raspberry Pi OS**
  - 13 Trixie: `arm64`, `armhf`.
  - 12 Bookworm: `arm64`, `armhf`.
  - 11 Bullseye: `arm64`, `armhf`.
- **Red Hat Enterprise Linux**
  - 10: `amd64`, `arm64`.
  - 9: `amd64`, `arm64`.
  - 8: `amd64`, `arm64`.
  - 7: `amd64`.
- **Rocky Linux**
  - 10: `amd64`, `arm64`.
  - 9: `amd64`, `arm64`.
  - 8: `amd64`, `arm64`.
- **SUSE Linux Enterprise Server**
  - 16: `amd64`, `arm64`.
  - 15 SP4 и новее в ветке 15: `amd64`, `arm64`.
  - 12 SP4 и новее в ветке 12: `amd64`.
- **Ubuntu**
  - 26.04 Resolute: `amd64`, `arm64`.
  - 24.04 Noble: `amd64`, `arm64`.
  - 22.04 Jammy: `amd64`, `arm64`.
  - 20.04 Focal: `amd64`, `arm64`.
  - 18.04 Bionic: `amd64`, `i386`.
  - 16.04 Xenial: `amd64`, `i386`.

Идентификаторы дистрибутивов: `almalinux`, `amazonlinux`, `centos`, `debian`, `opensuse-leap`, `oraclelinux`, `raspios`, `rhel`, `rocky`, `sles`, `ubuntu`. Для SLES пути `15-sp4` и `12-sp4` обозначают минимальный поддерживаемый service pack; совместимость с последующими SP проверяется отдельно.

Канонические имена архитектур: `amd64` = x86-64, `arm64` = AArch64, `i386` = 32-bit x86, `armhf` = 32-bit ARM hard-float. Минимальные CPU/ISA и ABI фиксируются в документации приложения и проверяются на целевых системах. `armhf` не означает автоматическую совместимость со всеми поколениями ARM; `amd64` не означает, что бинарник можно собирать с `-march=native`.

Наличие старой ОС в матрице означает требование совместимости приложения, а не обещание обновлений безопасности самой ОС. Для EOL-систем нужны зафиксированные архивные build-окружения. Нельзя незаметно заменить старую ОС новой или понизить требования проверки.

Windows, macOS, Alpine/musl и другие цели можно добавлять отдельным расширением контракта; они не подменяют перечисленную обязательную Linux-матрицу.

## 3. Формат и расположение файлов

Для каждой комбинации ОС/версии/архитектуры обязателен `.tar.gz` с готовым исполняемым файлом. Архив не должен требовать компиляции на машине получателя. `.deb` и `.rpm` допускаются дополнительно; наличие пакетов ещё не делает bucket полноценным APT/YUM-репозиторием — для этого нужны отдельные подписанные метаданные репозитория.

```text
<project>/
  builds/
    build-1/
      manifest.json
      manifest.json.minisig
      SHA256SUMS
      linux/
        debian/12/amd64/node-agent-build-1-linux-debian-12-amd64.tar.gz
        debian/12/arm64/node-agent-build-1-linux-debian-12-arm64.tar.gz
        ubuntu/24.04/amd64/node-agent-build-1-linux-ubuntu-24.04-amd64.tar.gz
        ...остальные обязательные цели...
    build-2/
      ...
  channels/
    stable.json
```

Полный URL примера:

```text
https://releases.soneyko.ai/node-agent/builds/build-1/linux/debian/12/amd64/node-agent-build-1-linux-debian-12-amd64.tar.gz
```

Внутри архива:

```text
bin/node-agent
README.md
LICENSE
licenses/                  # лицензии встроенных сторонних компонентов
systemd/node-agent.service # если приложение является systemd-сервисом
config/node-agent.example.yaml
```

Обязательны `bin/<project>` и лицензия приложения. Остальные элементы добавляются по необходимости. Исполняемые файлы сохраняют executable bit. Архив не содержит абсолютных путей, `..`, device nodes, setuid/setgid-файлов или ссылок, выходящих за каталог распаковки. Распаковка не должна менять владельца на root по данным архива.

Релизы хранятся до явного вывода из эксплуатации; предыдущая рабочая версия не удаляется автоматической очисткой. Неиспользуемые незавершённые Build можно чистить отдельной проверяемой процедурой, но не `sync --delete` из release workflow.

## 4. Сборка и проверка совместимости

- Сборка выполняется в изолированном окружении целевого дистрибутива или с корректным закреплённым sysroot/toolchain. Нельзя считать бинарник, собранный на свежем Ubuntu, совместимым с CentOS 7 или Ubuntu 16.04.
- Для glibc проверяются минимальная версия, dynamic loader и внешние библиотеки. Статический бинарник допустим, если приложение и его лицензии это позволяют.
- Можно переиспользовать идентичные байты для нескольких целей только после реальной проверки совместимости каждой цели; каждый target всё равно присутствует в manifest и имеет свой путь.
- Для каждой цели обязательны проверка формата/архитектуры ELF, запуск `--version` или аналогичного self-test и функциональный smoke test. Cross-compilation без запуска не равна проверенной поддержке.
- Чужие архитектуры проверяются на native runners или корректном QEMU-окружении. Установка QEMU и target-окружений входит в build-скрипт/инфраструктуру проекта.
- Точные версии compiler/toolchain, sysroot и container digest фиксируются в проекте. Системы с коммерческими образами/репозиториями требуют законного доступа, а не подмены совместимым дистрибутивом без проверки.
- Node-based Actions выполняются на современном runner; старое target-окружение запускается внутри build-инструмента, а не как job container, неспособный запустить сам Action.

## 5. Manifest, подпись и каналы

`manifest.json` содержит полный список целей, номер Build, project slug, commit SHA, дату UTC, размер и SHA-256 каждого архива. Подписываются **точные байты** manifest, а не JSON после повторной сериализации. Отдельный `SHA256SUMS` — удобство для ручной проверки; авторитетные хеши находятся в подписанном manifest.

Сокращённый пример структуры (реальный manifest содержит всю матрицу):

```json
{
  "schema_version": 1,
  "project": "node-agent",
  "build": 1,
  "release": "Build 1",
  "tag": "build-1",
  "source_commit": "0123456789abcdef0123456789abcdef01234567",
  "published_at": "2026-11-14T00:00:00Z",
  "artifacts": [
    {
      "os": "linux",
      "distribution": "debian",
      "version": "12",
      "arch": "amd64",
      "path": "node-agent/builds/build-1/linux/debian/12/amd64/node-agent-build-1-linux-debian-12-amd64.tar.gz",
      "size": 12345678,
      "sha256": "0000000000000000000000000000000000000000000000000000000000000000"
    }
  ]
}
```

Значения примера иллюстративные, не данные существующего релиза.

`channels/stable.json` — небольшой атомарно заменяемый указатель:

```json
{
  "project": "node-agent",
  "build": 1,
  "manifest": "node-agent/builds/build-1/manifest.json",
  "signature": "node-agent/builds/build-1/manifest.json.minisig"
}
```

Указатель — только механизм обнаружения, **не доверенный источник кода**. Клиент обязательно проверяет подпись manifest, project, Build и выбранную цель. Публичный ключ доставляется заранее доверенным способом, не скачивается вместе с бинарником с того же зеркала. Подмена указателя не должна позволять установить другой проект или откатить уже известный номер Build.

Для защиты новых установок от длительной выдачи старых, но корректно подписанных релизов нужны bootstrap minimum Build и/или протокол со сроками действия подписанных метаданных, например TUF. Один возрастающий локальный счётчик не решает freeze-attack для новой машины.

## 6. Cloudflare R2 и кеширование

1. Создать bucket для релизов и подключить к нему custom domain `releases.soneyko.ai`. DNS-запись сама по себе не подключает bucket; домен добавляется в настройках R2.
2. Выбрать публичное чтение через custom domain либо отдельный механизм авторизации клиентов. Пример ниже предполагает публичные артефакты, не приватное распространение.
3. Публиковать через `https://<ACCOUNT_ID>.r2.cloudflarestorage.com`, region `auto`. В ключе объекта нет начального `/` и имени bucket.
4. Для `*/builds/*`: `Cache-Control: public, max-age=31536000, immutable`.
5. Для `*/channels/*`: `Cache-Control: no-store`, плюс Cache Rule **Bypass cache**. Это правило должно иметь приоритет над общим Cache Everything.
6. Не кешировать ошибки/404 публикации надолго. Если до публикации URL проверялся через CDN, инвалидировать отрицательный кеш или дождаться его истечения.
7. Публичный домен должен отдавать бинарники без HTML challenge, интерактивной авторизации и преобразования содержимого. Проверять Content-Type, длину и SHA-256 реальным скачиванием с домена.
8. Не использовать `r2.dev` как production URL. Ограничить токен конкретным bucket, не выдавать account-wide административные права.

Один общий write-key bucket, выданный нескольким репозиториям, **не изолирует их по project prefix**: скомпрометированный workflow может изменить чужие объекты. Для независимых границ доверия использовать отдельные buckets/учётные данные или центральный publisher, который проверяет проект и разрешённый prefix. Пример ниже предназначен только для доверенных репозиториев-публикаторов.

## 7. Переменные и секреты GitHub Organization

**Да, настройки организации доступны workflows разрешённых репозиториев.** Их не требуется копировать в каждый репозиторий. Это не переменные ОС: в Actions они читаются через `vars.NAME` и `secrets.NAME`.

В организации: **Settings → Secrets and variables → Actions → Variables / Secrets**. В Repository access выбрать `Selected repositories` и явно перечислить доверенные репозитории. `All repositories` допустимо для несекретных общих настроек; не выдавать write/signing secrets всем репозиториям без проверки их границы доверия.

### Organization Variables: несекретные настройки

- `R2_ACCOUNT_ID` — Cloudflare account ID.
- `R2_BUCKET` — bucket релизов.
- `RELEASES_BASE_URL` — `https://releases.soneyko.ai`.
- `RELEASE_SIGNING_PUBLIC_KEY` — публичный ключ minisign; пригоден для проверки CI и документирования bootstrap, но клиенты должны получить доверенную копию независимо.

### Organization Secrets: только доверенным publishers

- `R2_ACCESS_KEY_ID` — S3 access key с правами работы с нужным bucket.
- `R2_SECRET_ACCESS_KEY` — секретная часть S3 key.
- `RELEASE_SIGNING_KEY_B64` — base64 приватного **незашифрованного** ключа minisign. Base64 — кодирование, не шифрование; защита обеспечивается GitHub Secrets и ограничением доступа к jobs.

Ключ подписи генерируется вне CI: `minisign -G -W`. Приватный ключ не коммитится и не загружается в R2. Для разных границ доверия лучше разные ключи проекта, а не один общий ключ на всю организацию. При этом одноимённый repository/environment secret может перекрыть organization secret.

### Repository Variable

- `RELEASE_PROJECT` — постоянный slug конкретной программы, например `node-agent`. Не задавать один общий slug для всей организации.

### Важные особенности доступа

- Organization-level variables и secrets имеют политику доступа по репозиториям. Для private repositories на GitHub Free organization-level secrets/variables недоступны; нужен подходящий план либо repository-level настройки.
- Для одноимённых настроек более узкий уровень обычно имеет приоритет: environment → repository → organization. Environment values доступны только job с соответствующим environment; не рассчитывать на них при планировании матрицы до запуска job.
- Fork pull requests не должны получать release secrets. Не использовать `pull_request_target` для исполнения чужого build-скрипта с ключами публикации.
- В reusable workflow secrets передаются явно либо через `secrets: inherit` в разрешённом контексте. Сам факт наличия org secret не означает его автоматическое наследование вызываемым workflow.
- `GITHUB_TOKEN` создаётся GitHub для текущего репозитория. Для создания Release достаточно `contents: write` в publish job; PAT всей организации не нужен. Этот token не даёт произвольного доступа к другим private repositories.
- Ключи публикации не передаются build jobs. Publish job исполняется только из защищённой default branch. Защитить `.github/workflows/` и `ci/` через branch protection/CODEOWNERS; любой, кто может изменить trusted publisher, потенциально может использовать его secrets.
- Для повышения защиты назначить publish job GitHub Environment с required reviewers. Если нужен полностью автоматический выпуск, отсутствие ручного approval — осознанное решение, а не обход проверки.

## 8. Пример GitHub Actions

Ниже файлы, которые нужно разместить **в исходном репозитории приложения**. В этом репозитории они приведены как документация, не как включённый workflow.

### `ci/targets.py`: единая машиночитаемая матрица

```python
import json

TARGETS = {
    "almalinux": {"10": "amd64 arm64", "9": "amd64 arm64", "8": "amd64 arm64"},
    "amazonlinux": {"2023": "amd64 arm64", "2": "amd64 arm64"},
    "centos": {"10": "amd64 arm64", "9": "amd64 arm64", "8": "amd64 arm64", "7": "amd64"},
    "debian": {"13": "amd64 arm64", "12": "amd64 arm64", "11": "amd64", "10": "amd64 i386"},
    "opensuse-leap": {"16": "amd64 arm64", "15": "amd64 arm64"},
    "oraclelinux": {"10": "amd64 arm64", "9": "amd64 arm64", "8": "amd64 arm64", "7": "amd64"},
    "raspios": {"13": "arm64 armhf", "12": "arm64 armhf", "11": "arm64 armhf"},
    "rhel": {"10": "amd64 arm64", "9": "amd64 arm64", "8": "amd64 arm64", "7": "amd64"},
    "rocky": {"10": "amd64 arm64", "9": "amd64 arm64", "8": "amd64 arm64"},
    "sles": {"16": "amd64 arm64", "15-sp4": "amd64 arm64", "12-sp4": "amd64"},
    "ubuntu": {"26.04": "amd64 arm64", "24.04": "amd64 arm64", "22.04": "amd64 arm64", "20.04": "amd64 arm64", "18.04": "amd64 i386", "16.04": "amd64 i386"},
}

if __name__ == "__main__":
    print(json.dumps({"include": [
        {"distro": distro, "version": version, "arch": arch}
        for distro, versions in TARGETS.items()
        for version, arches in versions.items()
        for arch in arches.split()
    ]}, separators=(",", ":")))
```

### Контракт `ci/build-target.sh`

Этот **проектный** скрипт нужно написать и проверить для своего языка и зависимостей. Универсальной команды сборки всех программ нет.

```text
bash ci/build-target.sh <distro> <version> <arch> <build-number> <project-slug>
```

Скрипт обязан собрать программу из текущего checkout, проверить её в целевой среде и создать ровно один архив:

```text
out/<distro>/<version>/<arch>/<project>-build-<N>-linux-<distro>-<version>-<arch>.tar.gz
```

Например:

```bash
bash ci/build-target.sh debian 12 amd64 1 node-agent
# out/debian/12/amd64/node-agent-build-1-linux-debian-12-amd64.tar.gz
```

Exit 0 разрешён только после успешных tests и smoke. Внутреннюю архитектуру toolchain нужно сопоставлять явно: например, Go использует `386`, а путь релиза — `i386`. Для cgo/C/C++/Rust недостаточно просто поменять переменную архитектуры: нужны sysroot, loader и библиотеки целевой ОС.

### `.github/workflows/release.yml`

```yaml
name: Release
on:
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: release-${{ github.repository }}
  cancel-in-progress: false

jobs:
  plan:
    if: github.ref == format('refs/heads/{0}', github.event.repository.default_branch)
    runs-on: ubuntu-24.04
    outputs:
      matrix: ${{ steps.targets.outputs.matrix }}
    steps:
      - uses: actions/checkout@11d5960a326750d5838078e36cf38b85af677262 # v4
      - id: targets
        shell: bash
        run: echo "matrix=$(python3 ci/targets.py)" >> "$GITHUB_OUTPUT"

  build:
    needs: plan
    runs-on: ubuntu-24.04
    strategy:
      fail-fast: false
      max-parallel: 8
      matrix: ${{ fromJSON(needs.plan.outputs.matrix) }}
    env:
      PROJECT: ${{ vars.RELEASE_PROJECT }}
      BUILD: ${{ github.run_number }}
      DISTRO: ${{ matrix.distro }}
      VERSION: ${{ matrix.version }}
      ARCH: ${{ matrix.arch }}
    steps:
      - uses: actions/checkout@11d5960a326750d5838078e36cf38b85af677262 # v4
      - name: Build and test target
        shell: bash
        run: |
          set -euo pipefail
          [[ "$PROJECT" =~ ^[a-z0-9]+(-[a-z0-9]+)*$ ]]
          bash ci/build-target.sh "$DISTRO" "$VERSION" "$ARCH" "$BUILD" "$PROJECT"
          test -s "out/$DISTRO/$VERSION/$ARCH/$PROJECT-build-$BUILD-linux-$DISTRO-$VERSION-$ARCH.tar.gz"
      - uses: actions/upload-artifact@ea165f8d65b6e75b540449e92b4886f43607fa02 # v4
        with:
          name: target-${{ matrix.distro }}-${{ matrix.version }}-${{ matrix.arch }}
          path: out/
          if-no-files-found: error
          retention-days: 7

  publish:
    needs: [plan, build]
    runs-on: ubuntu-24.04
    permissions:
      contents: write
    env:
      PROJECT: ${{ vars.RELEASE_PROJECT }}
      BUILD: ${{ github.run_number }}
      R2_ACCOUNT_ID: ${{ vars.R2_ACCOUNT_ID }}
      R2_BUCKET: ${{ vars.R2_BUCKET }}
      RELEASES_BASE_URL: ${{ vars.RELEASES_BASE_URL }}
      RELEASE_SIGNING_PUBLIC_KEY: ${{ vars.RELEASE_SIGNING_PUBLIC_KEY }}
      AWS_ACCESS_KEY_ID: ${{ secrets.R2_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.R2_SECRET_ACCESS_KEY }}
      AWS_REQUEST_CHECKSUM_CALCULATION: when_required
      AWS_RESPONSE_CHECKSUM_VALIDATION: when_required
      GH_TOKEN: ${{ github.token }}
    steps:
      - uses: actions/checkout@11d5960a326750d5838078e36cf38b85af677262 # v4
      - uses: actions/download-artifact@d3f86a106a0bac45b974a628896c90dbdf5c8093 # v4
        with:
          pattern: target-*
          path: out/
          merge-multiple: true
      - name: Install publishing tools
        run: |
          sudo apt-get update
          sudo apt-get install -y minisign
          python3 -m venv "$RUNNER_TEMP/publisher"
          "$RUNNER_TEMP/publisher/bin/pip" install boto3
      - name: Publish immutable Build and promote stable
        env:
          RELEASE_SIGNING_KEY_B64: ${{ secrets.RELEASE_SIGNING_KEY_B64 }}
        shell: bash
        run: |
          set -euo pipefail
          umask 077
          key="$RUNNER_TEMP/release-signing.key"
          trap 'rm -f "$key"' EXIT
          printf '%s' "$RELEASE_SIGNING_KEY_B64" | base64 --decode > "$key"
          export RELEASE_SIGNING_KEY_FILE="$key"
          "$RUNNER_TEMP/publisher/bin/python" ci/publish.py
```

Закреплённые SHA Actions нужно обновлять проверяемыми PR. Для production CI также зафиксировать версии и хеши publishing dependencies в lock-файле проекта; установка `boto3` выше показывает минимальную настройку, а не воспроизводимый dependency lock. GitHub-hosted runner должен иметь `gh`; для другого runner установить его отдельно.

### `ci/publish.py`: подпись, загрузка, readback, GitHub Release, stable

Пример использует обычный `PutObject` и ограничивает каждый файл 128 MiB. Для больших артефактов нужен отдельный multipart uploader с условием create-only на **CompleteMultipartUpload**, а не только на начале загрузки.

```python
import datetime
import hashlib
import json
import os
from pathlib import Path
import re
import subprocess
import sys
import tempfile
import urllib.request

import boto3
from botocore.exceptions import ClientError


def digest(data):
    return hashlib.sha256(data).hexdigest()


def encode(value):
    return (json.dumps(value, ensure_ascii=False, sort_keys=True, indent=2) + "\n").encode()


project = os.environ["PROJECT"]
build = int(os.environ["BUILD"])
account = os.environ["R2_ACCOUNT_ID"]
bucket = os.environ["R2_BUCKET"]
base = os.environ["RELEASES_BASE_URL"].rstrip("/")
assert re.fullmatch(r"[a-z0-9]+(-[a-z0-9]+)*", project)
assert re.fullmatch(r"[0-9a-fA-F]{32}", account)
assert build > 0 and base.startswith("https://")
assert subprocess.check_output(["git", "rev-parse", "HEAD"], text=True).strip() == os.environ["GITHUB_SHA"]

tag = f"build-{build}"
prefix = f"{project}/builds/{tag}"
channel_key = f"{project}/channels/stable.json"
s3 = boto3.client("s3", endpoint_url=f"https://{account}.r2.cloudflarestorage.com", region_name="auto")

# Проверка текущего stable и ETag для compare-and-swap, не слепая перезапись.
try:
    old = s3.get_object(Bucket=bucket, Key=channel_key)
except ClientError as exc:
    if exc.response["ResponseMetadata"]["HTTPStatusCode"] != 404:
        raise
    channel_condition = {"IfNoneMatch": "*"}
else:
    previous = json.loads(old["Body"].read())
    assert previous["project"] == project and previous["build"] < build
    channel_condition = {"IfMatch": old["ETag"]}

ref = subprocess.run(["git", "ls-remote", "--exit-code", "origin", f"refs/tags/{tag}"], capture_output=True)
assert ref.returncode == 2, "Tag exists or remote lookup failed; do not reuse a Build"

matrix = json.loads(subprocess.check_output([sys.executable, "ci/targets.py"]))["include"]
assert len({(t['distro'], t['version'], t['arch']) for t in matrix}) == len(matrix)
artifacts = []
uploads = []
for target in matrix:
    distro, version, arch = (target[k] for k in ("distro", "version", "arch"))
    filename = f"{project}-{tag}-linux-{distro}-{version}-{arch}.tar.gz"
    local = Path("out") / distro / version / arch / filename
    assert local.is_file() and not local.is_symlink(), str(local)
    body = local.read_bytes()
    assert 0 < len(body) <= 128 * 1024 * 1024
    key = f"{prefix}/linux/{distro}/{version}/{arch}/{filename}"
    artifacts.append(dict(os="linux", distribution=distro, version=version, arch=arch,
                          path=key, size=len(body), sha256=digest(body)))
    uploads.append((local, key, body, "application/gzip"))
assert {p.resolve() for p in Path("out").rglob("*.tar.gz")} == {p.resolve() for p, *_ in uploads}

manifest = dict(schema_version=1, project=project, build=build, release=f"Build {build}", tag=tag,
                source_commit=os.environ["GITHUB_SHA"],
                published_at=datetime.datetime.now(datetime.timezone.utc).isoformat(), artifacts=artifacts)
release_dir = Path("release-metadata")
release_dir.mkdir(exist_ok=True)
manifest_file = release_dir / "manifest.json"
manifest_file.write_bytes(encode(manifest))
signature_file = release_dir / "manifest.json.minisig"
subprocess.run(["minisign", "-S", "-s", os.environ["RELEASE_SIGNING_KEY_FILE"],
                "-m", str(manifest_file), "-x", str(signature_file)], check=True)
subprocess.run(["minisign", "-V", "-P", os.environ["RELEASE_SIGNING_PUBLIC_KEY"],
                "-m", str(manifest_file), "-x", str(signature_file)], check=True)
checksums = release_dir / "SHA256SUMS"
checksums.write_text("".join(f"{a['sha256']}  {a['path'][len(prefix)+1:]}\n" for a in artifacts))
for local, content_type in [(manifest_file, "application/json"),
                            (signature_file, "text/plain"), (checksums, "text/plain")]:
    uploads.append((local, f"{prefix}/{local.name}", local.read_bytes(), content_type))

# Create-only на самом write. Существующие объекты не перезаписываются даже при гонке.
for local, key, body, content_type in uploads:
    s3.put_object(Bucket=bucket, Key=key, Body=body, ContentType=content_type,
                  CacheControl="public, max-age=31536000, immutable", IfNoneMatch="*")
    received = s3.get_object(Bucket=bucket, Key=key)["Body"].read()
    assert digest(received) == digest(body), f"R2 readback mismatch: {key}"
    with urllib.request.urlopen(f"{base}/{key}", timeout=120) as response:
        assert digest(response.read()) == digest(body), f"Public readback mismatch: {key}"

# GitHub Release получает архивы и подписанные метаданные; tag указывает на исходный commit.
subprocess.run(["gh", "release", "create", tag, "--target", os.environ["GITHUB_SHA"],
                "--title", f"Build {build}", "--notes", f"Source: {os.environ['GITHUB_SHA']}\nManifest: {base}/{prefix}/manifest.json",
                *[str(local) for local, *_ in uploads]], check=True)
release = json.loads(subprocess.check_output(["gh", "release", "view", tag,
                                               "--json", "name,tagName,isDraft,assets"]))
assert release["name"] == f"Build {build}" and release["tagName"] == tag and not release["isDraft"]
assert {a["name"] for a in release["assets"]} == {p.name for p, *_ in uploads}

channel = encode(dict(project=project, build=build, manifest=f"{prefix}/manifest.json",
                      signature=f"{prefix}/manifest.json.minisig"))
s3.put_object(Bucket=bucket, Key=channel_key, Body=channel, ContentType="application/json",
              CacheControl="no-store", **channel_condition)
assert s3.get_object(Bucket=bucket, Key=channel_key)["Body"].read() == channel
with urllib.request.urlopen(f"{base}/{channel_key}", timeout=30) as response:
    assert response.read() == channel, "Channel cache mismatch"
print(f"Published Build {build}: {len(artifacts)} targets")
```

Если upload, readback или GitHub Release завершился ошибкой, workflow завершается ошибкой и не должен продвигать `stable`. При неоднозначном сетевом исходе записи channel нужно прочитать его фактическое значение: timeout не доказывает, что write не состоялся. Ошибка CDN-проверки после записи тоже не означает автоматический rollback. Не удалять объекты в попытке повторно использовать тот же Build; запускать новый run с новым номером.

Все архивы должны иметь уникальные имена, иначе GitHub Release assets будут конфликтовать. Пример проверяет доступность публичных файлов и состав Release; для более строгой приёмки дополнительно скачивать GitHub assets и сверять их хеши.

## 9. Установка и обновление на машинах

- Доверенный bootstrap устанавливает public key, ожидаемый project slug, target и начальный minimum Build. Не выполнять скачанный `curl | bash` без проверки подписи.
- Конфиги: `/etc/<project>/`; данные: `/var/lib/<project>/`; версии: `/opt/<project>/releases/build-N/`; активная версия: `/opt/<project>/current`.
- `systemd timer` или другой updater читает `stable.json`, скачивает manifest и подпись, проверяет подпись minisign и ожидаемые project/Build/target, затем размер и SHA-256 архива.
- Обновления сериализуются локальным lock. Архив распаковывается в новый временный каталог с проверкой путей и типов файлов, не поверх работающей версии.
- После проверки выполняется атомарное переключение `current`, controlled restart и health check. При неуспехе возвращается сохранённая рабочая версия и сервис запускается снова.
- Последний доверенный номер Build хранится отдельно от активной версии: локальный rollback не разрешает зеркалу снова предлагать старые Build как новые. Обновление на тот же номер не выполняется.
- Конфиги и данные не перезаписываются. Откат бинарника не откатывает БД; несовместимые миграции требуют отдельного плана и backup.
- Сначала canary-группа, затем остальные машины. Центральный bucket не должен означать одновременный рестарт всего парка.

## 10. Приёмка нового проекта

- [ ] Закреплены slug, source repository, владелец workflow и правило `Build N` / `build-N`.
- [ ] Все обязательные distro/version/architecture targets присутствуют в `ci/targets.py`.
- [ ] `ci/build-target.sh` реально собирает и проверяет каждую цель, а не только переименовывает один бинарник.
- [ ] Проверены лицензии, CPU/ABI/glibc и отсутствие секретов в артефактах.
- [ ] Настроены R2 custom domain, TLS, read policy, Cache Rules и права bucket.
- [ ] Organization variables/secrets видны только нужным publishers; отдельный project slug задан в repository variable.
- [ ] Actions/toolchains/dependencies закреплены, default branch и CI-файлы защищены.
- [ ] Полная матрица зелёная; GitHub Release имеет точное имя `Build N` и правильный commit.
- [ ] Архивы, manifest, подпись и checksums доступны по ожидаемым URL; readback совпадает.
- [ ] `stable` обновлён последним и не отдаётся из устаревшего кеша.
- [ ] Проверены установка, canary upgrade, неуспешный health check, rollback и сохранность конфигов/данных.

Изменение этой спецификации не означает, что перечисленные CI, R2 или updater уже настроены. Их включение и фактическая поддержка целей подтверждаются отдельными результатами сборки и испытаний.
