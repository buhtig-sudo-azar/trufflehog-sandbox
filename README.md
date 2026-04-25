# trufflehog-sandbox
# TruffleHog Sandbox

Учебный репозиторий для экспериментов с TruffleHog — инструментом поиска секретов (пароли, токены, ключи API) в Git‑репозиториях.

## Установка TruffleHog в Codespaces / Linux

```bash
mkdir -p "$HOME/.local/bin"

curl -sSfL https://raw.githubusercontent.com/trufflesecurity/trufflehog/main/scripts/install.sh \
  | sh -s -- -b "$HOME/.local/bin"

echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

trufflehog --version
Сканирование локального репозитория
bash
cd /workspaces/trufflehog-sandbox

trufflehog git file://. --results=verified,unknown --fail
JSON‑вывод:

bash
trufflehog git file://. --results=verified,unknown --json --fail
Сканирование репозитория по ссылке (GitHub)
bash
trufflehog git https://github.com/buhtig-sudo-azar/trufflehog-sandbox.git \
  --results=verified,unknown \
  --fail
Добавление учебного «секрета»
bash
cat > config.txt << 'EOF'
DEBUG=true
AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
EOF

git add config.txt
git commit -m "Add fake AWS keys"

trufflehog git file://. --results=verified,unknown --fail
Пример GitHub Actions workflow
text
name: secret-scan

on:
  push:
    branches: [ main ]
  pull_request:

jobs:
  trufflehog:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
        with:
          fetch-depth: 0

      - name: Install TruffleHog
        run: |
          curl -sSfL https://raw.githubusercontent.com/trufflesecurity/trufflehog/main/scripts/install.sh | sh -s -- -b /usr/local/bin

      - name: Run TruffleHog
        run: |
          trufflehog git file://. --results=verified,unknown --fail
