### Практическая работа "SIEM-система Wazuh"

---

#### О SIEM-системах

**SIEM** (Security Information and Event Management) – это система управления информацией и событиями безопасности, которая помогает организациям централизованно собирать, анализировать и управлять логами и событиями из различных серверов, сетей, приложений и баз данных. Основная цель SIEM-систем – выявлять потенциальные угрозы, уязвимости и инциденты безопасности в режиме реального времени, предоставляя аналитическую информацию для реагирования.

### Основные функции SIEM-систем

1. **Сбор данных** и событий с разных устройств и систем в организации.
2. **Анализ и корреляция событий**: объединение информации из разных источников, выявление подозрительных паттернов и взаимосвязей.
3. **Мониторинг и оповещение**: обнаружение аномалий и отправка уведомлений или тревог в случае подозрительной активности.
4. **Хранение данных** для выполнения аналитики, расследования инцидентов и соответствия требованиям регуляторов.
5. **Отчеты и соответствие стандартам**: подготовка отчетов для аудита и соблюдения стандартов безопасности (например, PCI DSS, GDPR).

Коммерческих SIEM-систем много, большинство из них являются закрытым и платным ПО, что делает их неудобными для самостоятельного развертывания и изучения.

[Wazuh](https://wazuh.com) – это бесплатная, открытая SIEM-платформа с функциями IDS (Intrusion Detection System, система обнаружения вторжений), которая активно развивается и становится популярной благодаря своей гибкости, поддержке широкого набора функций и легкости интеграции с другими системами. Она построена на базе Elastic Stack (ранее известного как ELK Stack) и предоставляет широкие возможности для мониторинга и управления безопасностью в реальном времени.

#### Задача и архитектура тестового стенда

Для выполнения лабораторной работы необходимо самостоятельно [собрать и настроить](https://documentation.wazuh.com/current/installation-guide/index.html) тестовый стенд, состоящий из следующих компонент:

1. **Control plane:** Linux-хост с серверной частью Wazuh.
    1. `wazuh.manager`
    2. `wazuh.indexer`
    3. `wazuh.dashboard`
2. **Клиенты:** Один или более хост **Linux**, на котором [установлен и настроен Wazuh agent](https://documentation.wazuh.com/current/installation-guide/wazuh-agent/index.html). Допустим любой *серверный дистрибутив без GUI-интерфейса* - **Ubuntu**, **Debian**, **Arch**, **Void**.

> [!TIP]
> Количество физических хостов в данном стенде не имеет значения, но вероятнее всего физический хост будет один. В таком случае, вот один из проверенных способов построить необходимую архитектуру:
> 1. Серверную часть развернуть с помощью `docker compose` прямо на самом физическом хосте.
> 2. Для клиентов подготовить обычные виртуальные машины под любым гипервизором, рекомендуется KVM/QEMU.

Высокоуровневая архитектура стенда такова:

> ![Wazuh architecture](https://documentation.wazuh.com/current/_images/poc-lab-env-arch1.png)
> Тут есть `Windows endpoint` - в данной работе он не нужен.

После подготовки Control plane и Linux-хостов, необходимо привязать агенты на клиентах к серверной части стенда. Установка, настройка и привязка агента описана в [документации Wazuh](https://documentation.wazuh.com/current/user-manual/agent/index.html).

На данном этапе, тестовый стенд должен представлять из себя функционирующую SIEM-систему из одного Command & Control Server и одного/нескольких клиентов, подключенных к нему. Дальнейший ход работы представляет из себя имитацию различных сбоев и атак на клиенты, с целью демонстрации функционала Wazuh и работоспособности настроенного стенда.

Список инцидентов для демонстрации:

1. [Brute-force атака на SSH сервер](https://documentation.wazuh.com/current/proof-of-concept-guide/detect-brute-force-attack.html)
2. [Мониторинг событий Docker](https://documentation.wazuh.com/current/proof-of-concept-guide/monitoring-docker.html)
3. [Обнаружение подозрительных процессов](https://documentation.wazuh.com/current/proof-of-concept-guide/detect-unauthorized-processes-netcat.html)
4. [Мониторинг целостности файловой системы](https://documentation.wazuh.com/current/proof-of-concept-guide/poc-file-integrity-monitoring.html)

#### Ход работы

> Всё администрирование серверной части и клиентов должно выполняться строго в командной строке. Доступ к виртуальным машинам осуществлять по SSH.
> Единственные GUI-инструменты, которые допускается использовать - это те, что являются частью самого Wazuh. Всё остальное - строго в CLI.

**ВАЖНО:** При установке и настройке всех хостов, следует дать пользователям и машинами уникальные имена - к примеру `ivanov_bsbo-01-22@ivanov_ii_siem0`, или что-то похожее. Для демонстрации работоспособности стенда необходимо будет прикрепить скриншоты из Wazuh Dashboard, а события там содержут множество полей с `hostname` и `username`, относящихся к инциденту.

> Dashboard серверной части доступен по `https://localhost:443` и выглядит так:
> ![[wazuh_dashboard.png]]
> Первоначальные alerts скорее всего начнут появляться даже то того, как будет настроен и подключен первый агент.

После добавления первого агента, он должен появиться в разделе Endpoints.

![[wazuh_endpoints.png]]

> События можно по-разному визуализировать, например так выглядит диаграмма `количество событий / уровень`:
> ![[wazuh_visualization.png]]

###### Инцидент №1: Brute-force атака на SSH сервер

Для демонстрации данной атаки никакой дополнительной конфигурации производить не нужно - достаточно настроенного Linux-endpoint. С любой другой машины (например, с физического хоста) начните Brute-force атаку, например с помощью [`hydra`](https://github.com/vanhauser-thc/thc-hydra). 

> Запуск атаки на ВМ по адресу `192.168.122.221`:
> ![[hydra_attack.png]]

На dashboard'е сразу должны начать появляться множество Medium Severity предупреждений о неудачных попытках входа в систему. В разделе Threat Hunting можно подробнее рассмотреть эти угрозы:

![[wazuh_threat_hunting.png]]

> 2 "Level 12 or above" предупреждения вызваны мной специально. Рассмотрим один из них подробнее во вкладке Events:
> ![[wazuh_ssh_event.png]]
> Тут детально описан весь инцидент. Как можно видеть из поля `full_log`, некий пользователь (я) успешно вошел в систему по паролю.

###### Инцидент №2: Мониторинг событий Docker

`todo!()`

#### Случайные записи

###### Установка агента на Ubuntu server 22.04

```bash
# run as root!

# Добавление репозитория с пакетами.
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | gpg --no-default-keyring --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg --import && chmod 644 /usr/share/keyrings/wazuh.gpg
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] https://packages.wazuh.com/4.x/apt/ stable main" | tee -a /etc/apt/sources.list.d/wazuh.list
apt-get update

# Установка агента.
WAZUH_MANAGER="192.168.122.221" apt-get install wazuh-agent

# Включение сервиса агента.
systemctl daemon-reload
systemctl enable wazuh-agent
systemctl start wazuh-agent
```
