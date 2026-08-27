# WireGuard Easy Server

[English](README.en.md) | **Русский**

Развёртывание [wg-easy](https://github.com/wg-easy/wg-easy) — сервера WireGuard VPN с веб-интерфейсом управления — на основе Docker Compose, дополненное обфускацией трафика [AmneziaWG](https://github.com/amnezia-vpn/amneziawg-linux-kernel-module) и реверс-прокси [Caddy](https://caddyserver.com/), который обеспечивает автоматический HTTPS для панели управления.

## Архитектура

| Компонент | Назначение | Внешние порты |
|-----------|------------|---------------|
| `wg-easy` | Сервер WireGuard/AmneziaWG и интерфейс управления | `443/udp` (точка подключения VPN) |
| `caddy` | Реверс-прокси с автоматическими сертификатами Let's Encrypt | `80/tcp`, `443/tcp` |

Ключевые решения:

- Панель управления не публикуется напрямую и доступна только через Caddy по HTTPS. Сертификаты выпускаются и продлеваются автоматически. Доменное имя задаётся в файле `Caddyfile`; далее в этом документе в качестве примера используется `vpn.example.com`.
- Точка подключения VPN слушает `443/udp`. На этом порту обфусцированный трафик AmneziaWG неотличим от QUIC (HTTP/3) для простых DPI-фильтров, а сам порт практически никогда не блокируется. Конфликта с Caddy нет: он занимает `443/tcp`.
- Порт, заданный в панели wg-easy, меняет реальный `ListenPort` интерфейса внутри контейнера, поэтому проброс портов в `compose.yaml` публикует тот же порт контейнера: `443:443/udp`. При несовпадении проброса и настройки панели клиенты не могут выполнить рукопожатие.
- `OVERRIDE_AUTO_AWG=awg` принудительно включает реализацию AmneziaWG: при отсутствии модуля ядра контейнер завершается с явной ошибкой, а не откатывается незаметно на обычный WireGuard.

## Требования

- Linux-сервер с Docker и плагином Docker Compose.
- DNS-запись типа `A` для выбранного домена (`vpn.example.com`), указывающая на публичный IP-адрес сервера (должна существовать до первого запуска, иначе выпуск сертификата завершится ошибкой).
- Открытые входящие порты на файрволе: `80/tcp`, `443/tcp` и `443/udp`.
- Установленный на хосте модуль ядра AmneziaWG (см. ниже).

## Пошаговая установка

### Шаг 1. DNS и сеть

- Создаётся DNS-запись типа `A` для выбранного домена (`vpn.example.com`), указывающая на публичный IP-адрес сервера. Запись должна распространиться до первого запуска — проверяется командой `nslookup vpn.example.com`.
- На файрволе сервера и в панели хостера (security group) открываются входящие порты `80/tcp`, `443/tcp` и `443/udp`. Порт `443/udp` открывается отдельной строкой — правило для TCP на него не распространяется.

### Шаг 2. Docker

При отсутствии Docker установка выполняется официальным скриптом:

```bash
curl -fsSL https://get.docker.com | sudo sh
```

Проверка: `docker compose version` выводит версию плагина Compose.

### Шаг 3. Модуль ядра AmneziaWG

Модуль устанавливается на хосте по инструкции из раздела [«Установка модуля ядра AmneziaWG»](#установка-модуля-ядра-amneziawg). Перед продолжением подтверждается загрузка модуля:

```bash
sudo modprobe amneziawg
lsmod | grep amneziawg
```

### Шаг 4. Файлы проекта

В рабочем каталоге на сервере (например, `~/wg-easy`) размещаются два файла:

- `compose.yaml` — полное содержимое приведено в разделе [«Файлы»](#файлы);
- `Caddyfile` — пример приведён в разделе [«Конфигурация реверс-прокси»](#конфигурация-реверс-прокси); плейсхолдер `vpn.example.com` заменяется на реальное доменное имя.

### Шаг 5. Запуск

```bash
docker compose up -d
```

В течение нескольких секунд Caddy получает сертификат Let's Encrypt, после чего панель управления открывается по адресу `https://vpn.example.com`. Ход выпуска сертификата при необходимости отслеживается командой `docker logs caddy -f`.

### Шаг 6. Первичная настройка wg-easy

При первом открытии панели создаётся учётная запись администратора. Затем в разделе *Admin Panel → General* указываются:

- **Host**: `vpn.example.com`;
- **Port**: `443`.

Эти значения попадают в поле `Endpoint` генерируемых клиентских конфигураций.

### Шаг 7. Подключение клиента

1. В панели создаётся клиент; конфигурация скачивается файлом или считывается QR-кодом.
2. На устройстве устанавливается приложение [AmneziaWG](https://docs.amnezia.org/documentation/amnezia-wg/) или AmneziaVPN (стандартный клиент WireGuard несовместим — см. раздел [«Диагностика»](#диагностика)).
3. Конфигурация импортируется в приложение, туннель включается.

Успешное подключение подтверждается появлением рукопожатия: в панели у клиента отображается время последнего handshake, на сервере то же видно в выводе `docker exec wg-easy awg show` (строки `latest handshake` и `transfer`).

## Установка модуля ядра AmneziaWG

Модуль устанавливается на хост-системе, а не внутри контейнера. Контейнер загружает его через смонтированный каталог `/lib/modules` и capability `SYS_MODULE` — оба уже настроены в `compose.yaml`.

### Ubuntu

Предварительно должны быть включены репозитории `deb-src` (пакет собирает модуль из исходников ядра): в Ubuntu 24.04+ для этого раскомментируется `deb-src` в строке `Types:` файла `/etc/apt/sources.list.d/ubuntu.sources`; в более ранних выпусках — строки `deb-src` в `/etc/apt/sources.list`.

```bash
sudo apt install -y software-properties-common python3-launchpadlib gnupg2 linux-headers-$(uname -r)
sudo add-apt-repository ppa:amnezia/ppa
sudo apt-get update
sudo apt-get install -y amneziawg
```

Пакет использует DKMS, поэтому при обновлении ядра модуль пересобирается автоматически.

### Другие дистрибутивы

Инструкции для Debian и других дистрибутивов, а также сборка из исходников описаны в оригинальной документации: [amnezia-vpn/amneziawg-linux-kernel-module](https://github.com/amnezia-vpn/amneziawg-linux-kernel-module).

### Проверка

```bash
sudo modprobe amneziawg
lsmod | grep amneziawg
```

Непустой вывод второй команды подтверждает, что модуль загружен.

## Развёртывание

```bash
docker compose up -d
```

При первом запуске Caddy получает сертификат Let's Encrypt для настроенного домена (занимает несколько секунд), после чего панель управления становится доступной по адресу `https://vpn.example.com`.

При первичной настройке wg-easy (или позднее в разделе *Admin Panel → General*) указываются следующие значения, чтобы генерируемые клиентские конфигурации содержали корректную точку подключения:

- **Host**: `vpn.example.com` (настроенный домен)
- **Port**: `443` (внешний UDP-порт)

Изменение значения **Port** меняет также `ListenPort` интерфейса внутри контейнера, поэтому проброс в `compose.yaml` должен публиковать именно этот порт контейнера. После любого изменения порта клиентские конфигурации скачиваются заново.

Клиентские приложения должны поддерживать AmneziaWG (приложения AmneziaWG или AmneziaVPN); стандартный клиент WireGuard не может выполнить рукопожатие с обфусцированным сервером. Конфигурации, генерируемые wg-easy, уже содержат необходимые параметры обфускации (`Jc`, `Jmin`, `Jmax`, `S1`, `S2`, `H1`–`H4`).

## Конфигурация реверс-прокси

Вся конфигурация Caddy состоит из одного блока. Пример файла `Caddyfile`:

```
vpn.example.com {
	reverse_proxy wg-easy:51821
}
```

Единственное значение, требующее изменения, — доменное имя. Выпуск и продление сертификатов, терминация TLS и перенаправление HTTP→HTTPS выполняются Caddy автоматически и дополнительных директив не требуют.

## Файлы

- [`compose.yaml`](compose.yaml) — описание сервисов (wg-easy, Caddy, сети, тома).
- `Caddyfile` — конфигурация реверс-прокси (см. пример выше).

<details>
<summary><code>compose.yaml</code> (полное содержимое)</summary>

```yaml
volumes:
  etc_wireguard:
  caddy_data:
  caddy_config:

services:
  wg-easy:
    environment:
     - EXPERIMENTAL_AWG=true
     - OVERRIDE_AUTO_AWG=awg

    image: ghcr.io/wg-easy/wg-easy:15
    container_name: wg-easy
    networks:
      wg:
        ipv4_address: 10.42.42.42
        ipv6_address: fdcc:ad94:bacf:61a3::2a
    volumes:
      - etc_wireguard:/etc/wireguard
      - /lib/modules:/lib/modules:ro
    ports:
      - "443:443/udp"
    restart: unless-stopped
    # Встроенный healthcheck образа использует `wg show`, который не видит
    # amneziawg-интерфейсы, поэтому контейнер всегда помечается unhealthy
    healthcheck:
      test: ["CMD-SHELL", "awg show | grep -q interface || exit 1"]
      interval: 60s
      timeout: 5s
      retries: 3
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
      # - NET_RAW # ⚠️ Uncomment if using Podman
    sysctls:
      - net.ipv4.ip_forward=1
      - net.ipv4.conf.all.src_valid_mark=1
      - net.ipv6.conf.all.disable_ipv6=0
      - net.ipv6.conf.all.forwarding=1
      - net.ipv6.conf.default.forwarding=1

  caddy:
    image: caddy:2
    container_name: caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
      - caddy_config:/config
    networks:
      - wg

networks:
  wg:
    driver: bridge
    enable_ipv6: true
    ipam:
      driver: default
      config:
        - subnet: 10.42.42.0/24
        - subnet: fdcc:ad94:bacf:61a3::/64
```

</details>

## Диагностика

### Нет рукопожатия (handshake)

Список проверок в порядке убывания вероятности, составленный по итогам реальной отладки этой конфигурации:

1. **Соответствие проброса портов и порта интерфейса.** Порт контейнера в секции `ports:` должен совпадать с портом, заданным в панели wg-easy (он определяет `ListenPort` интерфейса). Текущее состояние проверяется командой `docker exec wg-easy awg show` (строка `listening port`) и сравнивается с пробросом в выводе `docker ps`.
2. **Клиентское приложение.** Стандартный клиент WireGuard несовместим с обфусцированным сервером AmneziaWG; требуются приложения AmneziaWG или AmneziaVPN.
3. **Устаревшая клиентская конфигурация.** После смены порта в панели ранее скачанные конфигурации сохраняют старый `Endpoint` и подлежат повторному скачиванию.
4. **Файрвол провайдера.** `443/udp` в большинстве облачных security group открывается отдельно от `443/tcp`. Молчание `sudo tcpdump -ni any udp port 443` во время попытки подключения означает, что пакеты до сервера не доходят вовсе.
5. **Модуль ядра.** `lsmod | grep amneziawg` на хосте; при `OVERRIDE_AUTO_AWG=awg` отсутствие модуля приводит к явной ошибке контейнера.

### Контейнер в статусе `unhealthy`

Встроенный healthcheck образа выполняет стандартный `wg show`, который не видит интерфейсы AmneziaWG, поэтому контейнер постоянно помечается как `unhealthy` при полностью рабочем туннеле. Это ложная тревога, на работу VPN она не влияет.

В приведённом `compose.yaml` проблема устранена переопределением healthcheck на утилиту `awg`, присутствующую в образе:

```yaml
    healthcheck:
      test: ["CMD-SHELL", "awg show | grep -q interface || exit 1"]
      interval: 60s
      timeout: 5s
      retries: 3
```

По той же причине диагностика внутри контейнера выполняется командой `awg show`, а не `wg show`.

## Примечания

- Сертификаты хранятся в томе `caddy_data` и переживают пересоздание контейнеров.
- Конфигурация WireGuard и данные пиров хранятся в томе `etc_wireguard`.
- Некоторые сети (гостиницы, отдельные мобильные операторы) блокируют UDP полностью; выбор порта в этом случае не помогает, и в таких условиях потребуется транспорт поверх TCP.

## Ссылки

- [Документация wg-easy: AmneziaWG](https://wg-easy.github.io/wg-easy/v15.3/advanced/config/amnezia/)
- [Модуль ядра AmneziaWG для Linux](https://github.com/amnezia-vpn/amneziawg-linux-kernel-module)
- [Документация Caddy](https://caddyserver.com/docs/)
