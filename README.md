# Получение команд от Яндекс.Станции в Home Assistant
Компонент генерирует события в Home Assistant на конкретные голосовые команды, адресованные Яндекс.Станции (или колонкам с Алисой).
Как это работает: вы говорите "Алиса, сделай действие" и в Home Assistant появляется событие с текстом "сделай действие",
именем колонки и комнаты. Такие события очень удобно использовать в автоматизациях.

Компонент адресован продвинутым пользователям, которые не боятся поредактировать YAML, и является продолжателем функции
"Получение команд от станции" из компонента [Yandex.Station](https://github.com/AlexxIT/YandexStation#получение-команд-от-станции)
от [AlexxIT](https://github.com/AlexxIT).

Основные преимущества и отличия:
* **Автоматическое** создание/удаление/синхронизация сценариев в УДЯ (без перезагрузки HA!)
* Понятная и однозначная переменная `text` в событиях (вместо `Сделай громче на 0???!!!`)
* Возможно удалить все сценарии из УДЯ одним вызовом сервиса `yandex_station_intents.clear_scenarios`

**Компонент "Yandex.Station Intents" никак не связан с компонентом "Yandex.Station" и его авторами!**

- [Похожие компоненты](#похожие-компоненты)
- [Установка](#установка)
- [Настройка](#настройка)
  - [Список фраз (интенты)](#список-фраз-интенты)
  - [Обработка событий](#обработка-событий)
  - [Режим синхронизации](#режим-синхронизации)
  - [Режимы работы (для продвинутых)](#режимы-работы-для-продвинутых)
- [Вопросы и ответы](#вопросы-и-ответы)
  - [Зачем это всё, если можно отдать скрипты через Yandex Smart Home?](#зачем-это-всё-если-можно-отдать-скрипты-через-yandex-smart-home)
  - [Как управлять светом и другими штуками?](#как-управлять-светом-и-другими-штуками)
- [Миграция с YAML интентов Yandex.Station](#миграция-с-yaml-интентов-yandexstation)
  - [Удаление всех сценариев](#удаление-всех-сценариев)
- [Благодарности](#благодарности)


## Похожие компоненты
* [Yandex.Station](https://github.com/AlexxIT/YandexStation): управляет Яндекс.Станцией (воспроизведение, громкость, синтез текста), так же позволяет получать команды от Яндекс.Станции (с ручным созданием сценариев и в чуть менее удобной форме).
* [YandexDialogs](https://github.com/AlexxIT/YandexDialogs): обрабатывает любые фразы, адресованные Алисе, через NLU (Natural Language Processing). При использовании обязательно называть имя навыка, например "Алиса, попроси мой_навык сделать что нибудь".
* [Yandex Smart Home](https://github.com/dmitry-k/yandex_smart_home): позволяет управлять устройствами, подключенными к Home Assistant через Алису и веб-интерфейс. Используйте его для "Алиса, включи лампочку зелёным на 50% яркости" или "Алиса, прибавь громкость на телевизоре".


## Установка
**Способ 1:** [HACS](https://hacs.xyz/)

HACS > Интеграции > 3 точки (правый верхний угол) > Пользовательские репозитории > URL: `dext0r/ha-yandex-station-intents`, Категория: `Интеграция` > Нажать `Добавить` > Подождать > Выбрать `Yandex.Station Intents` в списке новых репозиториев > Нажать `Скачать`

**Способ 2:**

Вручную скопируйте папку `custom_components/yandex_station_intents` из [latest release](https://github.com/dext0r/ha-yandex-station-intents/releases/latest) в директорию `/config/custom_components`

После установки перезапустите Home Assistant.


## Настройка
* Добавьте интеграцию в Home Assistant: `Настройки` > `Устройства и службы` > `Интеграции` > Нажать `Добавить интеграцию` > `Yandex.Station Intents` (если интеграции нет в списке - обновите страницу)
* Авторизуйтесь в Яндексе, следуя подсказкам мастера. Вы можете использовать данные авторизации из компонента Yandex.Station, если он уже установлен
* Убедитесь, что Яндекс Станция добавлена в одну из комнат умного дома (через приложение Дом с Алисой)
* Настройте [список фраз](#список-фраз-интенты) или [мигрируйте](#миграция-с-yaml-интентов-yandexstation) с yaml интентов компонента Yandex.Station
* Перезагрузите YAML конфигурацию через `Панель разработчика` > `YAML`, в `Настройки` > `Система` > `Журнал сервера` появятся ошибки если что-то пошло не так


### Список фраз (интенты)
Интеграция работает с заранее сформированным списком фраз, на основе которых будут автоматически созданы сценарии в [УДЯ](https://yandex.ru/quasar/iot). Активационные фразы могут содержать **только** кириллицу, цифры и пробелы.

Настройка выполняется через основной файл конфигурации [configuration.yaml](https://www.home-assistant.io/docs/configuration/). Пример конфигурации:
```yaml
yandex_station_intents:
  intents:
    Как дела:                         # (1), символ : обязателен
    Кто нибудь дома: Сейчас проверю   # (2)
    Время ужинать:                    # (3)
      extra_phrases:                  # альтернативные фразы, максимум три
        - Давай кушать
        - Давай ужинать
        - Время ужина
    Не выключай свет в прихожей:      # (4)
      extra_phrases:
        - Не выключай свет в коридоре
      say_phrase: "{{ ['Договорились', 'Хорошо', 'Я тебя услышала', 'Оки-доки']|random }}"
    Давай попьем чаю:                 # (5)
      say_phrase: Отличная идея, сейчас включу свет на кухне
      execute_command: Включи свет на кухне
    Очень холодно:                    # (6)
      execute_command: Прибавь температуру кондиционера на 1 градус в {{ event.room }}
    Точная температура в комнате:     # (7)
      say_phrase: "Точная температура {{ sensor('sensor.room_temperature') }} в {{ event.room }}"
```

В данном случае интеграция автоматически создаст в УДЯ семь сценариев, каждый из которых начинается с символов `---`. **Не удаляйте эти символы и не модифицируйте никак название!** По ним компонент понимает, что это его сценарий и в случае необходимости синхронизирует/удалит его.

Дополнительные параметры `say_phrase`, `extra_phrases`, `execute_command` являются необязательными и могут использоваться в любых вариациях.

Как работает:
1. Срабатывает от `Алиса, как дела`, генерирует событие с `text: Как дела`, колонка ничего не скажет в ответ
2. Срабатывает от `Алиса, кто нибудь дома`, генерирует событие с `text: Кто нибудь дома`, колонка, которая нас услышала ответит `Сейчас проверю`
3. Срабатывает от `Алиса, время кушать` (или `Алиса, время ужина`, или `Алиса, давай ужинать` и т.п.), генерирует событие с `text: Время ужинать`, колонка ничего не скажет в ответ
4. Срабатывает от `Алиса, не выключай свет в прихожей` (или `Алиса, не выключай свет в коридоре`), генерирует событие с `text: Не выключай свет в прихожей`, колонка, которая нас услышала ответит случайной фразой из списка
5. Срабатывает от `Алиса, давай попьем чаю`, генерирует событие с `text: Давай попьем чаю`, колонка, которая нас услышала ответит `Отличная идея, сейчас включу свет на кухне`, после этого колонка выполнит команду `Включи свет на кухне`
6. Срабатывает от `Алиса, очень холодно`, генерирует событие с `text: Очень холодно`, колонка выполнит команду `Прибавь температуру кондиционера на 1 градус в КОМНАТА` (вместо `КОМНАТА` будет подставлена комната, в которой находится колонка)
7. Срабатывает от `Алиса, точная температура в комнате`, генерирует событие с `text: Точная температура в комнате`, колонка, которая нас услышала ответит вычисленным шаблоном из `say_phrase`

### Обработка событий
После того как колонка услышит ключевую фразу в Home Assistant сгенерируется событие `yandex_intent` с параметрами:
* `text`: Основная фраза (смотрите примеры)
* `entity_id`: ID колонки, которая услышала фразу активации (только `mode: websocket`)
* `room`: Комната, в которой находится колонка (только `mode: websocket`)

Пример обработки:
```yaml
automation:
  - alias: Кто нибудь дома
    trigger:
      - platform: event
        event_type: yandex_intent
        event_data:
          text: Кто нибудь дома  # пример (2)
    action:
      - service: media_player.play_media
        target:
          entity_id: '{{ trigger.event.data.entity_id }}'  # ответит колонка, которая услышала "Алиса, кто нибудь дома"
        data:
          media_content_type: text
          media_content_id: Я не знаю, ха-ха

  - alias: Как дела
    trigger:
      - platform: event
        event_type: yandex_intent
        event_data:
          text: Как дела  # пример (1)
          room: Кухня     # сработает, если спросили колонку в комнате "Кухня"
    action:
      - service: media_player.play_media
        target:
          entity_id: '{{ trigger.event.data.entity_id }}'
        data:
          media_content_type: text
          media_content_id: Здесь в комнате {{ trigger.event.data.room }} всё отлично!
```

### Режим синхронизации
По умолчанию сценарии синхронизируются с УДЯ автоматически при запуске Home Assistant. Это поведение можно отключить добавлением в конфигурацию опции `autosync: false`. После этого сценарии будут синхронизированы **только** при перезагрузке YAML конфигурации компонента со страницы `Панель разработчика` > `YAML` или через сервис `yandex_station_intents.reload`.

Пример:
```yaml
yandex_station_intents:
  autosync: false
  intents:
    Тест:
```

### Режимы работы (для продвинутых)
Интеграция поддерживает два режима работы: `websocket` (по-умолчанию) и `device`. Режим задаётся через параметр `mode` в конфигурации (менять можно в любой момент).

Режим `websocket` (по-умолчанию):
* Для работы требуется настроенная интеграция [Yandex.Station](https://github.com/AlexxIT/YandexStation)
* Постоянное подключение к серверам Яндекса, не требует интеграцию `Yandex Smart Home`
* Невозможно активировать событие из интерфейса УДЯ или голосом на телефоне, активация возможна только через колонку
* Позволяет получать `entity_id` и `room` колонки, которая услышала активационную фразу
* Использует недокументированную функцию УДЯ и в теории может быть отключен Яндексом

Режим `device` (условно-устаревший):
* Требует установленный компонент `Yandex Smart Home`, в фильтрах необходимо разрешить `media_player.yandex_station_intents`
* Можно активировать событие кнопкой в интерфейсе или голосом на телефоне
* **Невозможно** определить колонку, которая услышала фразу, отсутствуют параметры `entity_id` и `room` в событиях
* Будет работать всегда, так как активация происходит через управление виртуальным устройством в УДЯ
* В этом режиме неподдерживаются:
  * Параметр `execute_command`
  * Шаблоны в `say_phrase`

## Вопросы и ответы
### Зачем это всё, если можно отдать скрипты через Yandex Smart Home?
1. При активации скриптов Алиса много болтает ("хорошо", "сделала" и т.п.). Болтовню можно отключить только для всего умного дома целиком, что не всегда удобно
2. Невозможно достоверно определить колонку, которая активировала скрипт (только с некоторой вероятностью через мониторинг `alice_state: BUSY`)
3. События **очень удобно** использовать в качестве триггера в автоматизациях

### Как управлять светом и другими штуками?
Используйте интеграцию [Yandex Smart Home](https://github.com/dmitry-k/yandex_smart_home)

## Миграция с YAML интентов Yandex.Station
1. Установите и настройте интеграцию `Yandex.Station Intents` (YAML пока не меняйте)
2. Удалите сценарии, которые были созданы компонентом Yandex.Station. Если в УДЯ нет вручную созданных сценариев - воспользуйтесь сервисом [`yandex_station_intents.clear_scenarios`](#удаление-всех-сценариев), в противном случае - удалите сценарии вручную.
3. Убедитесь, что в УДЯ не осталось сценариев, созданных компонентом `Yandex.Station`
4. В YAML конфигурации измените `yandex_station` на `yandex_station_intents` (при условии, что у вас в `yandex_station` есть только блок `intents`)
5. Перезагрузите YAML конфигурацию `Yandex.Station Intents` сценарии будут созданы автоматически
6. Дождитесь появления всех сценариев в УДЯ (30-60 секунд), посмотрите Журнал Сервера на наличие ошибок
7. Перезагрузите Home Assistant, убедитесь, что ошибок по-прежнему нет
8. Проверьте работу интентов, изменения в автоматизациях не потребуются

### Удаление всех сценариев
Компонент позволяет удалить **абсолютно все** сценарии из УДЯ через сервис `yandex_station_intents.clear_scenarios`. Будут удалены в том числе и сценарии, созданные вручную.

Для удаления вызовите сервис через `Панель разработчика` > `Сервисы`:
```yaml
service: yandex_station_intents.clear_scenarios
data:
  confirm: Я действительно хочу удалить ВСЕ сценарии из УДЯ
```

## Благодарности
* [AlexxIT](https://github.com/AlexxIT) за компонент Yandex.Station и разрешение использовать из него код для авторизации в Яндексе
