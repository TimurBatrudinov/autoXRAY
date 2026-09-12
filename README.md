# autoXRAY

Автоматическая установка XRay на VPS: скрипт разворачивает ядро XRay, генерирует ключи во всех протоколах, настраивает сайт-заглушку (маскировку), HTTPS-сертификат Let's Encrypt и создаёт страницу с конфигурациями для пользователей.

## Требования

- VPS на Ubuntu/Debian, доступ по root
- **Домен** с одной A-записью, указывающей на IP вашего VPS (подойдёт любой DNS-сервис, включая бесплатные). Дождитесь распространения DNS перед установкой.

## Установка

```bash
bash -c "$(curl -L https://raw.githubusercontent.com/TimurBatrudinov/autoXRAY/main/autoXRAY1.sh)" -- ваш.домен.com
```

Скрипт автоматически определяет страну VPS — конфиги называются по схеме **«флаг страны + протокол»** (например, `🇩🇪 VLESS`, `🇩🇪 HYSTERIA2`); тип транспорта (xhttp, gRPC и т.д.) виден в клиенте.

## Настраиваемые протоколы

| Протокол | Транспорт | Примечание |
|---|---|---|
| VLESS + REALITY | raw, flow Vision | основной, порт 443 |
| VLESS + REALITY | xhttp | ссылка печатается только в консоли — нужна для моста RU→EU |
| VLESS + TLS | raw / xhttp / WS / gRPC | порт 8443 |
| Hysteria2 | QUIC | порт 8080 |
| socks5 (mixed) | — | порт 10443, авторизация |

## Что получает пользователь

- **Страница конфигураций** `https://ваш.домен.com/<случайный-путь>.html` — ссылки, QR-коды, кнопки копирования;
- **JSON-подписка** — готовый конфиг клиента со встроенным роутингом (RU-сегмент напрямую, реклама блокируется);
- секции приложений **HAPP** и **incy** с кнопками роутинга GeoGaga.

Рекомендуемые клиенты: [HAPP](https://www.happ.su/main/ru) и [incy](https://incy.cc/) (iOS, Android, Windows, macOS, Linux).

## Мост RU → EU

Если российский провайдер режет зарубежные VPS, трафик можно завести через RU-сервер:

1. Установите `autoXRAY1.sh` на EU-VPS и скопируйте из консоли ссылку **VLESS — XHTTP REALITY** (для моста).
2. Установите мост на RU-VPS, передав ссылку аргументом:

```bash
bash -c "$(curl -L https://raw.githubusercontent.com/TimurBatrudinov/autoXRAY/main/autoXRAYselfRUbrEUxhttp.sh)" -- поддомен.ваш.домен.com "vless://ссылка-из-консоли-EU"
```

Можно передать несколько ссылок через пробел. Для каждой EU-ноды мост создаёт два конфига через себя (флаг страны RU-VPS) и один прямой (флаг страны ноды), балансировка — leastLoad.

## Обновление и удаление

```bash
# обновить Xray-ядро
bash -c "$(curl -sL https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ update
# полное удаление: xray, nginx-конфиг, сайт и сертификаты
systemctl stop xray && rm -rf /usr/local/etc/xray /var/www/ваш.домен.com
```

## Поддержать автора

Проект основан на [xVRVx/autoXRAY](https://github.com/xVRVx/autoXRAY) (GPL-3.0):
https://github.com/xVRVx/autoXRAY
