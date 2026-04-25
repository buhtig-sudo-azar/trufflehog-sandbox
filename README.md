Сайт [https://devtrends.ru/go/trufflesecurity-trufflehog]

TruffleHog + mitmproxy: перехват HTTP‑трафика и поиск секретов
Цель: перехватывать HTTP/HTTPS‑трафик браузера через mitmproxy, сохранять ответы в файлы и сканировать их TruffleHog’ом на наличие секретов (пароли, токены, API‑ключи и т.п.).

1. Предусловия
Среда:

Kali Linux.

Установлены:

mitmproxy (даёт mitmdump),
​

curl, python3,

Firefox.

Установка TruffleHog в $HOME/.local/bin через install‑скрипт:

bash
mkdir -p "$HOME/.local/bin"
curl -sSfL https://raw.githubusercontent.com/trufflesecurity/trufflehog/main/scripts/install.sh \
  | sh -s -- -b "$HOME/.local/bin"

echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc

trufflehog --version
Если версия вывелась — всё ок, TruffleHog готов.

2. Подготовка папок и скрипта mitmproxy
2.1. Папка под ответы
bash
mkdir -p "$HOME/http_responses/download"
2.2. Скрипт save_responses.py
bash
cat > "$HOME/http_responses/save_responses.py" << 'EOF'
from mitmproxy import http
import os
import hashlib

OUTPUT_DIR = os.path.expanduser("~/http_responses/download")

ALLOWED_CONTENT_TYPES = (
    "text/html",
    "text/plain",
    "application/json",
    "application/javascript",
    "text/javascript",
)

def response(flow: http.HTTPFlow) -> None:
    try:
        if not flow.response:
            return

        content_type = flow.response.headers.get("Content-Type", "")
        if content_type:
            content_type = content_type.split(";")[0].strip().lower()
            if content_type not in ALLOWED_CONTENT_TYPES:
                return

        body = flow.response.content or b""
        if not body:
            return

        url = flow.request.url

        os.makedirs(OUTPUT_DIR, exist_ok=True)

        h = hashlib.sha256(url.encode() + body).hexdigest()[:16]
        filename = f"{h}.txt"
        path = os.path.join(OUTPUT_DIR, filename)

        with open(path, "wb") as f:
            f.write(b"URL: " + url.encode() + b"\n\n")
            f.write(body)

    except Exception:
        pass
EOF
Проверка:

bash
ls "$HOME/http_responses"
# должен быть save_responses.py и папка download
3. Запуск mitmdump с этим скриптом
bash
mitmdump -s "$HOME/http_responses/save_responses.py" -p 8083
-s — подключаем наш скрипт.

-p 8083 — слушаем порт 8083.

Весь трафик, прошедший через этот порт, будет обработан скриптом и подходящие ответы будут записаны в ~/http_responses/download как .txt файлы.

Окно с mitmdump оставляем открытым на время захвата трафика.

4. Настройка Firefox на прокси 127.0.0.1:8083
Вариант без плагинов:

Firefox → Settings / Настройки.

Внизу: Network Settings / Настройки сети → кнопка Settings… / Настроить….

Выбрать Manual proxy configuration / Ручная настройка прокси.

Ввести:

HTTP Proxy: 127.0.0.1

Port: 8083

Поставить галку Use this proxy server for all protocols.

Нажать OK.

Теперь весь трафик Firefox идёт через mitmdump на 8083.
​

5. Установка сертификата mitmproxy (для HTTPS)
Убедись, что mitmdump запущен на 8083.

В Firefox (с включённым прокси) зайди на:

text
http://mitm.it
Выбери Linux, скачай mitmproxy-ca.pem.

Открой: Settings → Privacy & Security.

Внизу: раздел Certificates → кнопка View Certificates….

В окне сертификатов:

вкладка Authorities,

кнопка Import…,

выбрать mitmproxy-ca.pem,

поставить галку «Trust this CA to identify websites»,

нажать OK.

Теперь mitmproxy сможет MITM’ить HTTPS‑трафик Firefox.
​

6. Как именно работает скрипт с download
Для каждого HTTP‑ответа:

mitmproxy вызывает функцию response(flow).

Скрипт проверяет:

есть ли flow.response;

берёт заголовок Content-Type, режет всё после ; и приводит к lower;

сохраняет только если тип в списке:

text/html,

text/plain,

application/json,

application/javascript,

text/javascript.

Если тип не подходит (например, image/png, font/woff2) — ответ игнорируется.

Если подходит:

берётся body = flow.response.content;

если тело пустое — выход;

берётся url = flow.request.url;

создаётся папка ~/http_responses/download (если её нет);

считается hash = sha256(url + body), первые 16 символов — имя файла;

создаётся файл ~/http_responses/download/<hash>.txt с содержимым:

text
URL: https://example.com/api/...

<сырое тело ответа>
7. Сбор трафика и проверка файлов
Запущен mitmdump -s ~/http_responses/save_responses.py -p 8083.

Firefox использует прокси 127.0.0.1:8083.

Сертификат mitmproxy установлен.

Дальше:

Ходишь по нужным сайтам/веб‑приложениям/API.

Проверяешь, что файлы создаются:

bash
ls "$HOME/http_responses/download"
# должны появляться файлы вида: 1a2b3c4d5e6f7a8b.txt

head -n 20 "$HOME/http_responses/download/любой_файл.txt"
8. Сканирование сохранённых ответов TruffleHog’ом
Так как всё лежит в ~/http_responses/download, сканируем именно эту папку.

8.1. Обычный вывод
bash
trufflehog filesystem "$HOME/http_responses/download" --results=verified,unknown --fail
filesystem — режим сканирования файловой системы.
​

Каталог — ~/http_responses/download.

--results=verified,unknown — показывать только подтверждённые и потенциальные секреты.

--fail — вернуть ненулевой код выхода, если секреты найдены (удобно для обёртки/CI).
​

8.2. JSON‑вывод
bash
trufflehog filesystem "$HOME/http_responses/download" --results=verified,unknown --json --fail
--json — вывод в JSON, удобно парсить jq, складывать в отчёты, отправлять в ELK/Grafana и т.д.

9. Как понимать результат TruffleHog
В конце запуска будет строка вида:

text
finished scanning {"chunks": ..., "bytes": ..., "verified_secrets": X, "unverified_secrets": Y, ...}
verified_secrets — сколько секретов TruffleHog смог подтвердить через API (рабочие ключи/токены).

unverified_secrets — сколько потенциальных секретов, требующих ручной проверки.

Если оба нули — в текущих перехваченных ответах ничего похожего на секреты нет.
Если X/Y > 0 — смотри основной вывод / JSON: там будет тип секрета, файл (~/http_responses/download/…), фрагмент значения и контекст.

10. Итоговый чек‑лист под этот скрипт
mkdir -p ~/http_responses/download

save_responses.py лежит в ~/http_responses/save_responses.py и пишет в ~/http_responses/download.

Запуск:

bash
mitmdump -s ~/http_responses/save_responses.py -p 8083
Firefox → прокси 127.0.0.1:8083.

http://mitm.it → Linux → импорт CA в Firefox → доверить.

Ходишь по целевым сайтам/API.

Проверяешь: ls ~/http_responses/download — есть .txt файлы.

Запуск TruffleHog:

bash
trufflehog filesystem ~/http_responses/download --results=verified,unknown --fail
Шпаргалка по TruffleHog (другие режимы)
TruffleHog умеет сканировать не только файлы/логи, но и git, Docker, S3 и удалённые репозитории.

A. Git‑репозитории (локальные)
Скан текущего локального git‑репо (код + история коммитов):

bash
cd /path/to/repo
trufflehog git file://. --results=verified,unknown --fail
git — режим сканирования git‑репозитория.

file://. — указание на локальный репозиторий в текущем каталоге.

B. GitHub / GitLab (организации и репо)
Через Docker (пример для GitHub‑организации):

bash
docker run --rm -it -v "$PWD:/pwd" \
  trufflesecurity/trufflehog:latest \
  github --org=your_org --results=verified,unknown --fail
Требуется GitHub‑токен (обычно в переменных окружения).

Аналогично есть подкоманды для GitLab, Bitbucket и т.д.

C. Docker‑образы
Скан Docker‑образа на секреты внутри слоёв:

bash
trufflehog docker my-image:latest --results=verified,unknown --fail
Полезно, если кто‑то положил креды в ENV, конфиги, внутри образа.

D. S3‑бакеты
Скан AWS S3:

bash
trufflehog s3 s3://my-bucket-name --results=verified,unknown --fail
Нужен корректный AWS‑профиль / переменные с доступом.

Можно сканировать как отдельные бакеты, так и целые аккаунты (через конфиг).

E. Логи и прочие файлы
Любые логи (CI/CD, приложения, nginx и т.п.) можно скормить через filesystem:

bash
trufflehog filesystem /var/log/myapp --results=verified,unknown --fail
Работает так же, как с ~/http_responses/download, только источник другой.

