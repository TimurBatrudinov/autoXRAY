# CLAUDE.md — контекст проекта autoXRAY

## Что это

Bash-скрипты для развёртывания личного VPN на базе [Xray-core](https://github.com/XTLS/Xray-core). Целевая платформа — чистый Ubuntu/Debian с root. Репозиторий публичный: `TimurBatrudinov/autoXRAY` (форк проекта xVRVx/autoXRAY, лицензия GPL-3.0 — атрибуция сохранена). Скрипты при установке подтягивают вспомогательные файлы с ветки `main` **этого же** репозитория — **после правок локальных копий `test/*.sh` изменения не действуют, пока не запушены**. Все функциональные ссылки в скриптах должны указывать на `TimurBatrudinov/autoXRAY`; ссылка на xVRVx остаётся только как «Поддержать автора».

## Два основных скрипта и их связь

| Скрипт | Роль |
|---|---|
| `autoXRAY1.sh` | Автономная установка на EU-VPS: 7 протоколов + сайт-заглушка + страница конфигов. Вызов: `./autoXRAY1.sh домен.com` |
| `autoXRAYselfRUbrEUxhttp.sh` | Мост RU→EU. Вызов: `./autoXRAYselfRUbrEUxhttp.sh мост.домен.com "vless://ссылка-EU" ["vless://ссылка-EU2" …]`. RU-клиент подключается к RU-VPS, трафик балансируется на EU-ноды |

**Связь:** при установке `autoXRAY1.sh` печатает в консоль ссылку «VLESS — XHTTP REALITY (для моста)». Её администратор вручную передаёт аргументом в мостовой скрипт. На пользовательскую страницу и в подписку эта ссылка **не попадает** (сознательно).

## Архитектура (общая для обоих скриптов)

- **443 (REALITY)**: vless + raw/reality; «чужой» TLS форвардится в nginx через unix-сокет `/dev/shm/nginx.sock` (selfsteal — сайт-заглушка крутится на своём же домене). Тело xhttp-запросов уходит fallback-ом на внутренний inbound `127.0.0.1:3333`.
- **Сайт-заглушка**: генерируется внешним скриптом `test/gen_page3.sh` (22 случайных «корпоративных» темы, случайный 401-ответ от `/api/v1/authenticate`). Скрипты качают его с GitHub при установке.
- **autoXRAY1.sh, порт 8443 (TLS)**: vless raw/xhttp/ws/grpc — TLS терминируется Xray-ом, nginx слушает те же unix-сокеты (`nginxTLS.sock`, `nginx_h2.sock`), wss/grpc проксируются nginx-ом на `8400`/`8411`.
- **Hysteria2 на 8080**, **socks5 (mixed) на 10443** (`socksUser`/`socksPasw`), опционально **WARP** (socks5 40000).
- **Мост**: один общий серверный UUID, у которого 3-я группа (7–8 байты) переписывается hex-номером ноды → routing-правило `vlessRoute` маршрутизирует на outbound `proxy-N`. Исходящие EU-нод собираются из параметров входных vless-ссылок. Балансировщик `Super_Balancer` (leastLoad).
- **Подписка**: `$WEB_PATH/$path_subpage.json` — массив полных клиентских конфигов (функция `print_config` + шаблоны outbound-ов, склейка через `envsubst`). nginx отдаёт её с заголовками Happ (`profile-title`, `routing` — встроенный роутинг в base64).
- **Страница конфигов**: `$WEB_PATH/$path_subpage.html`, генерируется heredoc-ами в 3–4 аппенда. `$path_subpage` — 20 случайных символов (угадать URL нельзя).

## Ключевые соглашения

- **Нейминг конфигов**: «флаг страны VPS + протокол» — `🇩🇪 VLESS`, `🇩🇪 HYSTERIA2`. Тип транспорта в имя не входит (виден в клиенте); на HTML-странице транспорт показывается бейджем в строке конфига.
- **Флаг**: автоопределение при установке (`ip-api.com` → `ipinfo.io` → ручной ввод, дефолт `EU`). Хелпер `cc_to_flag` превращает 2-буквенный код в эмодзи (regional indicators). Мостовой скрипт берёт флаг каждой EU-ноды из ремарки её входной ссылки (первое слово, дефолт 🇪🇺).
- **`urlencode`** (в обоих скриптах): побайтовое percent-кодирование для фрагмента `#…` ссылок (`#$VLESS_NAME_ENC`). НЕ используйте `envsubst`-переменные внутри quoted-heredoc — они не раскроются.
- **Heredoc-этапы страницы**: первый `cat > … <<'EOF'` (quoted) — статические head/CSS/JS без раскрытия переменных; далее `cat >> … <<EOF` (unquoted) — динамика (`$DOMAIN`, `$subPageLink`, цикл по `CONFIGS_ARRAY`). В JS внутри unquoted-heredoc нельзя использовать `$` и бэктики.
- **`CONFIGS_ARRAY`**: элементы формата `Имя|Бейдж|Ссылка`; парсинг в цикле: `title=${item%%|*}; rest=${item#*|}; badge=${rest%%|*}; link=${rest#*|}`.
- **envsubst-поток** (только autoXRAY1.sh): переменные серверного конфига и клиентских JSON должны быть `export`-нуты (строка `export …` перед `cat << 'EOF' | envsubst`).
- **MTProto/telemt удалён** (сентябрь 2026). REALITY-target всегда `/dev/shm/nginx.sock`. Больше не возвращать.

## Внешние ресурсы на странице конфигов

- HAPP: deep-link `happ://add/<подписка>`, сайт https://www.happ.su/main/ru, роутинг GeoGaga — https://geogaga-happ.rf.gd/
- incy: https://incy.cc/ (клиент с открытым ядром, VLESS/Hysteria2/REALITY), роутинг GeoGaga — https://geogaga-incy.rf.gd/

## Карта файлов

```
autoXRAY1.sh                     # основная установка (правится в первую очередь)
autoXRAYselfRUbrEUxhttp.sh       # мост RU→EU
test/gen_page3.sh                # генератор сайта-заглушки (используется обоими скриптами через curl с GitHub main)
test/gen_page2.sh                # предыдущее поколение генератора (не используется основными скриптами)
test/autoXRAY1-test.sh,          # устаревшие рабочие копии основных скриптов (не синхронизируются)
test/autoXRAYselfRUbrEUxhttp-test.sh
test/warp-cf.sh, warp-readme.md  # установка WARP-cli (fscarmen) и её readme
old/                             # архивные версии скриптов; описание — old/oldScriptReadme.md
```

## Как проверять правки

1. `bash -n <скрипт>` — синтаксис (heredoc-ы легко ломаются).
2. Юнит-проверка хелперов: вынести `urlencode`/`cc_to_flag` во временный файл и прогнать `bash`-ом (флаги: `DE`→🇩🇪 = `f0 9f 87 a9 f0 9f 87 aa`; `urlencode "🇩🇪 VLESS"` → `%F0%9F%87%A9%F0%9F%87%AA%20VLESS`).
3. Превью страницы: `sed -n '<строки генерации HTML>p' скрипт.sh > /tmp/gen.sh`, затем запустить с заглушками переменных (`DOMAIN=…`, `FLAG=…`, `subPageLink=…`, `CONFIGS_ARRAY=(…)`, `WEB_PATH=/tmp/prev`) и открыть HTML в браузере.
4. Ссылки на мост: EXTRA-ссылка из консоли `autoXRAY1.sh` должна парситься мостом (ремарка «флаг VLESS» → `NODE_FLAG`).
