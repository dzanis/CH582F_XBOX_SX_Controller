# CH582 Central для геймпада Xbox Series X

Этот проект демонстрирует подключение микроконтроллера CH582 (в режиме BLE Central) к геймпаду Xbox Series X (SX).

Цель — установить защищенное BLE-соединение, подписаться на нотификации сервиса HID (Human Interface Device) и получать данные о положении стиков и нажатии кнопок.

## Ключевые особенности и решения

Подключение к геймпаду Xbox требует решения нескольких нетривиальных задач, не описанных в стандартных примерах SDK:

1.  **Безопасность (Bonding):** Сервис HID (`0x1812`) является защищенным. Геймпад не отдаст данные, пока не будет выполнена процедура **сопряжения (Pairing) и сохранения ключей (Bonding)**.
2.  **Проблема тайминга (`Write Error: 80`):** При первом сопряжении стек WCH вызывает ошибку `Write Error: 80` (Insufficient Authorization) при попытке подписаться на нотификации. Это происходит потому, что GATT-операция запускается **слишком рано**, до того как стек безопасности полностью стабилизировал зашифрованный канал.
3.  **Автоматическое восстановление:** Геймпад **автоматически восстанавливает** нотификации при повторном подключении к уже спаренному устройству, что делает повторное "Discovery" (поиск сервисов) ненужным и даже вредным.

Этот код использует "двухэтапную" логику подключения, чтобы обойти эти проблемы.

## 1\. Логика работы: Стратегия "Двух Подключений"

Наш код (`central.c`) реализует два разных сценария подключения:

### Этап 1: Первое подключение (Сопряжение и Сохранение)

1.  CH582 сканирует эфир, находит геймпад по HID UUID (`adv_contains_uuid16`) и подключается.
2.  Чип принудительно запускает сопряжение (см. `DEFAULT_PAIRING_MODE` в `central.c`).
3.  После успешного обмена ключами срабатывает событие `GAPBOND_PAIRING_STATE_BOND_SAVED`.
4.  В этот момент мы **сохраняем MAC-адрес** геймпада в EEPROM.
5.  Мы **один раз** запускаем процедуру поиска сервисов (`START_SVC_DISCOVERY_EVT`).
6.  **(Ожидаемая ошибка)**: При попытке подписки (`START_WRITE_CCCD_EVT`) стек WCH вернет `Write Error: 80`, так как канал еще не стабилизировался.
7.  *Результат Этапа 1:* Устройства спарены, адрес сохранен, но нотификации не работают. Геймпад ищет (быстро моргает).

### Этап 2: Повторное подключение (Рабочий режим)

1.  Пользователь нажимает кнопку "Connect" на геймпаде **второй раз**.
2.  CH582, который либо перезапустился, либо все еще сканирует, видит знакомый MAC-адрес (`PeerAddrDef`, загруженный из EEPROM) и немедленно подключается (`GAP_DEVICE_DISCOVERY_EVENT`).
3.  Срабатывает событие `GAP_LINK_ESTABLISHED_EVENT`.
4.  Срабатывает событие `GAPBOND_PAIRING_STATE_BONDED` (подтверждение, что используется старый ключ).
5.  **(Ключевое решение):** Мы **не запускаем** `START_SVC_DISCOVERY_EVT` повторно.
6.  Геймпад, видя, что к нему подключился доверенный (спаренный) партнер, **автоматически восстанавливает** шифрование и **автоматически начинает слать нотификации**, так как он запомнил нашу подписку с первого (неудачного) раза.
7.  Данные стиков начинают поступать в `ATT_HANDLE_VALUE_NOTI`.

## 2\. Ключевые изменения в коде (`central.c`)

### 2.1. Активация сопряжения

Мы принудительно включаем сопряжение и отключаем MITM (PIN-код), так как геймпад использует "Just Works".

```c
// central.c - Глобальные константы
#define DEFAULT_PAIRING_MODE                GAPBOND_PAIRING_MODE_INITIATE
#define DEFAULT_MITM_MODE                   FALSE
#define DEFAULT_BONDING_MODE                TRUE

// central.c - Central_Init()
// ... (Настройка GAPBondMgr_SetParameter) ...
```

### 2.2. Сохранение адреса и запуск Discovery (Только один раз)

Логика запуска поиска сервисов находится **только** в `GAPBOND_PAIRING_STATE_BOND_SAVED`. Это гарантирует, что она выполнится лишь один раз за подключение к xbox контроллеру.

```c
// central.c - centralPairStateCB()
else if(state == GAPBOND_PAIRING_STATE_BOND_SAVED)
{
    if(status == SUCCESS)
    {
        PRINT("Bond save success\n");

        // Сохраняем адрес для будущих подключений
        EEPROM_ERASE(0, EEPROM_BLOCK_SIZE);
        EEPROM_WRITE(0,  PeerAddrDef, B_ADDR_LEN);
        
        // Запускаем быстрое моргание (индикация настройки)
        setLedBlink(LED_BLINK_FAST_MS);
        
        // Запускаем поиск сервисов (который, как мы ожидаем, не 
        // сможет подписаться из-за ошибки 80, но это нормально)
        tmos_start_task(centralTaskId, START_SVC_DISCOVERY_EVT, DEFAULT_SVC_DISCOVERY_DELAY);
    }
}
```

### 2.3. Пропуск Discovery при повторном подключении

При установке соединения и подтверждении старого ключа (`BONDED`) мы намеренно **ничего не делаем**.

```c
// central.c - centralEventCB()
case GAP_LINK_ESTABLISHED_EVENT:
{
    // ...
    // Initiate service discovery
    // tmos_start_task(centralTaskId, START_SVC_DISCOVERY_EVT, ...); // ЗАКОММЕНТИРОВАНО! иначе будет ошибка
    // ...
}

// central.c - centralPairStateCB()
else if(state == GAPBOND_PAIRING_STATE_BONDED)
{
    if(status == SUCCESS)
    {
        PRINT("Bonding success\n");
        // Ничего не запускаем. Ждем автоматического восстановления.
    }
}
```

### 2.4. Индикация и состояние "Готов"

LED используется для индикации трех состояний:

1.  **Медленное моргание:** (`LED_BLINK_SLOW_MS`) Идет сканирование.
2.  **Быстрое моргание:** (`LED_BLINK_FAST_MS`) Идет настройка (после `BOND_SAVED`).
3.  **Постоянное горение:** (`LED_BLINK_OFF`) Соединение установлено и стабильно.

Финальный переход в состояние "Готов" (постоянный свет) происходит при событии `GAP_LINK_PARAM_UPDATE_EVENT`. Это событие доказывает, что соединение полностью стабилизировалось.

```c
// central.c - centralEventCB()
case GAP_LINK_PARAM_UPDATE_EVENT:
{
    PRINT("Param Update...\n");
    setLedBlink(LED_BLINK_OFF); // Отключаем моргание, LED перейдет в ON
}

// central.c - Central_ProcessEvent()
if (events & START_LED_BLINK_EVT)
{
    GPIOA_InverseBits(GPIO_Pin_8); // Мигаем
    if(ledBlinkPeriod > 0)
    {
        tmos_start_task(centralTaskId, START_LED_BLINK_EVT, ledBlinkPeriod); 
    }
    else
    {
        GPIOA_ResetBits(GPIO_Pin_8); // LED ON (ResetBits = 0 = Включено)
    }
}
```

## 3\. Чтение данных стиков

Данные приходят в `centralProcessGATTMsg` -\> `ATT_HANDLE_VALUE_NOTI`. Они передаются в коллбэк `gamepadInputCB`.

Пример парсинга (для `gamepadInputCB`):

```c
void GamepadInputCallback(uint8_t *data, uint16_t len)
{
    PRINT("Gamepad Input Report Data: ");
    for (uint16_t i = 0; i < len; i++) {
        PRINT("%02X ", data[i]);
    }
    PRINT("\n");
}

// ... в main() или init:
Central_RegisterGamepadInputCallback(GamepadInputCallback);
```