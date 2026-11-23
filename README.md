# FreePBX Click-to-Call PHP Script

[![PHP Version](https://img.shields.io/badge/PHP-7.0%2B-blue.svg)](https://www.php.net/)
[![FreePBX Compatibility](https://img.shields.io/badge/FreePBX-14%2B-green.svg)](https://www.freepbx.org/)
[![Asterisk Compatibility](https://img.shields.io/badge/Asterisk-13%2B-orange.svg)](https://www.asterisk.org/)

---

# Документация (русский)

## Описание

Простой PHP-скрипт для серверов FreePBX и Asterisk: веб-API Click-to-Call. По HTTP-запросу с параметрами внутреннего номера (`exten`) и вызываемого номера (`number`) скрипт через Asterisk Manager Interface (AMI) сначала звонит на указанный внутренний номер, затем соединяет абонента с целевым номером. Ответ — JSON с признаком успеха, описанием ошибки и типом технологии (PJSIP или CHAN_SIP). Реализована проверка формата параметров и ограничение доступа по IP (только частные сети). Скрипт размещается в веб-каталоге PBX и вызывается как `click.php`.

Адаптация [скрипта Alisson Pelizaro](https://github.com/alissonpelizaro/Asterisk-Click-to-Call) с валидацией входных данных, JSON-ответами, поддержкой PJSIP и расширенной обработкой ошибок.

## Содержание

- [Возможности](#возможности)
- [Требования](#требования)
- [Установка](#установка)
- [Настройка](#настройка)
- [Использование](#использование)
- [Документация API](#документация-api)
- [Безопасность](#безопасность)
- [Устранение неполадок](#устранение-неполадок)
- [Удалённая работа](#удалённая-работа)
- [История версий](#история-версий)
- [Лицензия](#лицензия)
- [Благодарности](#благодарности)

## Возможности

- **Простая интеграция**: быстрое подключение к существующей связке FreePBX и Asterisk.
- **Ответ в JSON**: удобная интеграция с CRM, порталами и другими приложениями.
- **CHAN_SIP и PJSIP**: автоматическое определение технологии внутреннего номера.
- **Проверка данных и IP**: валидация параметров и доступ только из частных сетей.
- **Асинхронный Originate**: вызовы ставятся в очередь AMI без блокировки HTTP-ответа.
- **Подробные ошибки**: понятные сообщения в поле `Description`.
- **Шаблон Caller ID**: настраиваемый отображаемый номер при исходящем вызове.

## Требования

Перед установкой убедитесь, что выполнены условия:

- **PHP**: 7.0 и выше, расширения `json` и `sockets`
- **FreePBX**: 14 и выше (проверялось на v14)
- **Asterisk**: 13 и выше (проверялось на v18)
- **Сеть**: для безопасности скрипт рассчитан на обращение из локальной (частной) сети
- **AMI**: включён и настроен Asterisk Manager Interface

## Установка

1. **Загрузите скрипт**
   - Поместите файл `click.php` в веб-каталог, например:
     ```bash
     /var/www/html/
     ```

2. **Настройте учётные данные AMI**
   - Параметры берутся из:
     ```bash
     /etc/asterisk/manager.conf
     ```
   - Укажите логин и пароль AMI в массиве `$config` в начале `click.php`.

3. **Права доступа**
   - Веб-сервер должен иметь право читать и выполнять PHP-файл:
     ```bash
     chmod 644 /var/www/html/click.php
     chown www-data:www-data /var/www/html/click.php
     ```

4. **Проверка**
   - Откройте в браузере или выполните HTTP-запрос:
     ```
     http://YOUR.PBX.IP/click.php?exten=123&number=0612345678
     ```
   - В ответе должен быть JSON об успехе или об ошибке конфигурации.

## Настройка

Параметры задаются в массиве `$config` в начале `click.php`:

| Параметр | По умолчанию | Описание |
|----------|----------------|----------|
| `host` | 127.0.0.1 | Адрес AMI; обычно localhost, если скрипт на том же сервере, что FreePBX |
| `port` | 5038 | Порт AMI |
| `user` | admin | Имя пользователя AMI из `manager.conf` |
| `secret` | MYSECRETPASS | Пароль пользователя AMI |
| `callerIdTemplate` | CTR Plugin (%s) | Шаблон Caller ID; `%s` — вызываемый номер |
| `context` | from-internal | Контекст диалплана для исходящего вызова (стандарт FreePBX) |
| `waitTime` | 30 | Секунд ожидания ответа внутреннего номера |
| `priority` | 1 | Приоритет в диалплане |
| `maxRetry` | 2 | Зарезервировано в конфиге (в текущей версии не используется в коде) |

### Примеры изменений

Шаблон Caller ID «Исходящий на %s»:

```php
$config = [
    // ... other settings ...
    'callerIdTemplate' => 'Outbound Call to (%s)',
    // ... other settings ...
];
```

Увеличить время ожидания до 45 секунд:

```php
$config = [
    // ... other settings ...
    'waitTime' => 45,
    // ... other settings ...
];
```

## Использование

### Базовый сценарий

```http
http://YOUR.PBX.IP/click.php?exten=123&number=0612345678
```

- `123` — внутренний номер, который должен зазвонить первым.
- `0612345678` — номер, который наберёт АТС после ответа на внутреннем.

Сначала звонит указанный внутренний; после ответа FreePBX соединяет с заданным номером.

### Параметры запроса

| Параметр | Описание | Правила |
|----------|----------|---------|
| `exten` | Внутренний номер, который звонит первым | Только цифры 0–9 |
| `number` | Номер для дозвона после ответа | Цифры и опционально `+` в начале |

### Примеры вызова API

#### cURL

```bash
curl "http://YOUR.PBX.IP/click.php?exten=123&number=0612345678"
```

#### PHP

```php
$response = file_get_contents('http://YOUR.PBX.IP/click.php?exten=123&number=0612345678');
$result = json_decode($response, true);
if ($result['Success']) {
    echo "Call initiated successfully!";
} else {
    echo "Error: " . $result['Description'];
}
```

#### JavaScript / jQuery

```javascript
$.getJSON('http://YOUR.PBX.IP/click.php?exten=123&number=0612345678', function(data) {
    if (data.Success) {
        alert("Call initiated successfully!");
    } else {
        alert("Error: " + data.Description);
    }
});
```

## Удалённая работа

Скрипт удобен сотрудникам вне офиса, если внутренний номер переадресован на мобильный: звонок идёт через корпоративную АТС, а не с личного телефона.

### Как это работает

1. Сотрудник нажимает ссылку Click-to-Call в CRM, почте и т.п.
2. Скрипт инициирует вызов на внутренний номер сотрудника.
3. При переадресации звонок приходит на мобильный.
4. После ответа АТС набирает целевой номер.
5. Тарификация по правилам PBX, а не мобильного оператора.

### Пример: плагин Outlook

Интеграция с кастомным плагином Outlook (не входит в этот репозиторий): клик по номеру в письме → запрос к API → звонок на внутренний → переадресация на мобильный → соединение с выбранным номером.

**Важно:** по умолчанию доступ к скрипту разрешён только с частных IP. Для доступа из интернета нужны VPN в офисную сеть и/или доработка проверки IP и дополнительная авторизация.

## Документация API

HTTP API с ответами в формате JSON.

### Точка входа

```
GET /click.php
```

### Параметры

| Параметр | Обязательный | Описание |
|----------|--------------|----------|
| `exten` | Да | Внутренний номер, который звонит первым |
| `number` | Да | Номер для соединения после ответа |

### Формат ответа

| Поле | Тип | Описание |
|------|-----|----------|
| `Success` | boolean | Успешно ли поставлен в очередь вызов |
| `ValidInput` | boolean | Корректны ли входные параметры |
| `Description` | string | Текст результата или ошибки |
| `Technology` | string | PJSIP или SIP (при успехе) |
| `OriginateResponse` | string | Сырой ответ AMI (при успехе) |

### Примеры ответов

#### Успех

```json
{
  "Success": true,
  "ValidInput": true,
  "Description": "Extension 123 is calling 0612345678.",
  "Technology": "PJSIP",
  "OriginateResponse": "Response: Success\r\nMessage: Originate successfully queued\r\n\r\n"
}
```

#### Неверный формат номера

```json
{
  "Success": false,
  "ValidInput": false,
  "Description": "Invalid number format: abc123",
  "Technology": "",
  "OriginateResponse": ""
}
```

#### Неверный внутренний номер

```json
{
  "Success": false,
  "ValidInput": false,
  "Description": "Invalid extension format: abc",
  "Technology": "",
  "OriginateResponse": ""
}
```

#### Запрещённый IP

```json
{
  "Success": false,
  "ValidInput": false,
  "Description": "Unauthorized IP address: 203.0.113.1",
  "Technology": "",
  "OriginateResponse": ""
}
```

#### Ошибка аутентификации AMI

```json
{
  "Success": false,
  "ValidInput": false,
  "Description": "Authentication failed",
  "Technology": "",
  "OriginateResponse": ""
}
```

#### Не удалось определить технологию

```json
{
  "Success": false,
  "ValidInput": false,
  "Description": "Failed to retrieve technology for extension: 123",
  "Technology": "",
  "OriginateResponse": ""
}
```

#### Сбой инициации вызова

```json
{
  "Success": false,
  "ValidInput": false,
  "Description": "Call initiation failed",
  "Technology": "PJSIP",
  "OriginateResponse": "Response: Error\r\nMessage: Channel PJSIP/123 not available\r\n\r\n"
}
```

## Безопасность

### Проверка IP

Запросы с публичных адресов отклоняются:

```php
if (filter_var($_SERVER["REMOTE_ADDR"], FILTER_VALIDATE_IP, FILTER_FLAG_NO_PRIV_RANGE | FILTER_FLAG_NO_RES_RANGE)) {
    $result = setError($result, "Unauthorized IP address: %s", $_SERVER["REMOTE_ADDR"]);
}
```

Для доступа с конкретных внешних IP нужно изменить эту проверку (или использовать VPN и токен).

### Валидация параметров

- `exten` — только цифры
- `number` — цифры и опционально ведущий `+`

Рекомендуется также: вынести пароль AMI из веб-каталога, ограничить права пользователя AMI, добавить токен API и HTTPS.

## Устранение неполадок

### Внутренний не звонит

- **Регистрация абонента**:
  ```bash
  asterisk -rx "pjsip show endpoint <extension>"
  asterisk -rx "sip show peer <extension>"
  ```
- **Контекст диалплана** (`from-internal` по умолчанию):
  ```bash
  asterisk -rx "dialplan show from-internal"
  ```
- **Файрвол и NAT**:
  ```bash
  iptables -L -n
  ```

### Ошибки AMI

- **Учётные данные** — совпадение с `/etc/asterisk/manager.conf`:
  ```bash
  grep -A 10 "\[admin\]" /etc/asterisk/manager.conf
  ```
- **Права AMI-пользователя**:
  ```
  read = system,call,log,verbose,command,agent,user,config,dtmf,reporting,cdr,dialplan,originate
  write = system,call,log,verbose,command,agent,user,config,dtmf,reporting,cdr,dialplan,originate
  ```

### Определение технологии (PJSIP/SIP)

```bash
asterisk -rx "database show DEVICE"
```

### Отладка в CLI

```bash
asterisk -rvvvvv
```

```
core set verbose 5
```

### Логи

```bash
tail -f /var/log/asterisk/full
```

## История версий

### v1.0.1 (2026-11-23)

- Переименование основного скрипта в `click.php`
- Полная русская и английская документация в README
- Добавлен CHANGELOG.md, схема версий 1.0.x

### v1.0.0 (2023-04-01)

- Первый релиз
- PJSIP и CHAN_SIP
- JSON API, валидация, проверка IP

## Лицензия

Проект распространяется по лицензии MIT — см. файл LICENSE.

## Благодарности

Часть работы выполнена при поддержке [Contour — поставщик листового металла](https://www.contour.eu/en/).

Адаптация [Asterisk Click-to-Call](https://github.com/alissonpelizaro/Asterisk-Click-to-Call) Alisson Pelizaro.

---

# Documentation (English)

## Description

A simple and effective PHP script for FreePBX and Asterisk servers: a Click-to-Call web API. An HTTP request with extension (`exten`) and destination (`number`) uses the Asterisk Manager Interface (AMI) to ring the extension first, then connect the callee to the target number. Responses are JSON (success, errors, PJSIP/SIP technology). Input validation and private-network IP checks are included. Deploy as `click.php` under your PBX web root.

Adapted from [Alisson Pelizaro's Click-to-Call script](https://github.com/alissonpelizaro/Asterisk-Click-to-Call), with input validation, JSON responses, PJSIP support, and improved error handling.

## Table of Contents

- [Features](#features)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Usage](#usage)
- [API Documentation](#api-documentation)
- [Security Considerations](#security-considerations)
- [Troubleshooting](#troubleshooting)
- [Remote Work Usage](#remote-work-usage)
- [Version History](#version-history)
- [License](#license)
- [Credits](#credits)

## Features

- **Easy Integration**: Quickly integrate with your existing FreePBX and Asterisk setup.
- **JSON Response**: Provides clear JSON responses for easy integration with other applications.
- **CHAN_SIP and PJSIP Support**: Fully compatible with both CHAN_SIP and PJSIP technology.
- **Secure and Validated**: Includes input validation and IP address checks for enhanced security.
- **Asynchronous Call Origination**: Calls are originated asynchronously for better performance.
- **Detailed Error Handling**: Provides descriptive error messages for troubleshooting.
- **Technology Detection**: Automatically detects whether an extension uses PJSIP or SIP technology.
- **Customizable Caller ID**: Supports custom caller ID templates.

## Prerequisites

Before installing the Click-to-Call script, ensure your system meets the following requirements:

- **PHP**: Version 7.0 or higher with `json` and `sockets` extensions enabled
- **FreePBX**: Version 14 or higher (tested with v14)
- **Asterisk**: Version 13 or higher (tested with v18)
- **Network**: Script must be accessed from a local network (private IP range) for security
- **AMI Access**: Asterisk Manager Interface must be enabled and configured

## Installation

Follow these simple steps to install and configure the Click-to-Call script on your FreePBX server:

1. **Upload the Script**
   - Place the `click.php` file in your web directory:
     ```bash
     /var/www/html/
     ```

2. **Configure AMI Credentials**
   - Edit the script and set your AMI credentials found in:
     ```bash
     /etc/asterisk/manager.conf
     ```
   - Update the `$config` array in the script with your AMI username and password.

3. **Set Permissions**
   - Ensure the script has appropriate permissions so that the web server can execute the PHP script:
     ```bash
     chmod 644 /var/www/html/click.php
     chown www-data:www-data /var/www/html/click.php
     ```

4. **Verify Installation**
   - Test the script by accessing it through your web browser:
     ```
     http://YOUR.PBX.IP/click.php?exten=123&number=0612345678
     ```
   - You should receive a JSON response indicating success or any configuration issues.

## Configuration

The script includes several configuration options that can be customized to fit your specific needs. These options are defined in the `$config` array at the beginning of the script:

| Option | Default | Description |
|--------|---------|-------------|
| `host` | 127.0.0.1 | The IP address of the Asterisk Manager Interface (AMI). Usually localhost (127.0.0.1) if the script is running on the same server as FreePBX. |
| `port` | 5038 | The port number for the AMI. The default is 5038, but it may be different in your configuration. |
| `user` | admin | The username for AMI authentication. This should match a user defined in `/etc/asterisk/manager.conf`. |
| `secret` | MYSECRETPASS | The password for AMI authentication. This should match the password for the user defined in `/etc/asterisk/manager.conf`. |
| `callerIdTemplate` | CTR Plugin (%s) | The template for the caller ID. The `%s` will be replaced with the number being called. |
| `context` | from-internal | The dialplan context to use for the outgoing call. The default is `from-internal`, which is standard for FreePBX. |
| `waitTime` | 30 | The number of seconds to wait for the extension to answer before timing out. |
| `priority` | 1 | The dialplan priority to use for the outgoing call. |
| `maxRetry` | 2 | Reserved in config (not used in the current script version). |

### Example Configuration Changes

To change the caller ID template to show "Outbound Call to %s":

```php
$config = [
    // ... other settings ...
    'callerIdTemplate' => 'Outbound Call to (%s)',
    // ... other settings ...
];
```

To increase the wait time to 45 seconds:

```php
$config = [
    // ... other settings ...
    'waitTime' => 45,
    // ... other settings ...
];
```

## Usage

### Basic Usage

To initiate a call, simply open the following URL in your browser or call it via an HTTP request:

```http
http://YOUR.PBX.IP/click.php?exten=123&number=0612345678
```

- Replace `123` with the extension number you want to ring first.
- Replace `0612345678` with the phone number you wish to call.

The specified extension will ring first. Once answered, FreePBX will automatically dial the provided phone number.

### Parameter Details

| Parameter | Description | Validation Rules |
|-----------|-------------|------------------|
| `exten` | The extension that will ring first. This must be a valid extension configured in your FreePBX system. | Must contain only digits (0-9). |
| `number` | The phone number to call after the extension answers. This can be an internal extension, a local number, or an international number depending on your dialplan. | Must contain only digits (0-9) and optionally a leading plus sign (+). |

### API Usage Examples

#### Using cURL

```bash
curl "http://YOUR.PBX.IP/click.php?exten=123&number=0612345678"
```

#### Using PHP

```php
$response = file_get_contents('http://YOUR.PBX.IP/click.php?exten=123&number=0612345678');
$result = json_decode($response, true);
if ($result['Success']) {
    echo "Call initiated successfully!";
} else {
    echo "Error: " . $result['Description'];
}
```

#### Using JavaScript/jQuery

```javascript
$.getJSON('http://YOUR.PBX.IP/click.php?exten=123&number=0612345678', function(data) {
    if (data.Success) {
        alert("Call initiated successfully!");
    } else {
        alert("Error: " + data.Description);
    }
});
```

## Remote Work Usage

This Click-to-Call script is particularly useful for remote workers who wish to use their work PBX server for initiating calls. By configuring the personal extension to forward incoming calls to an employee's mobile number, remote workers can seamlessly use their work phone number instead of their personal mobile number.

### How It Works for Remote Workers

1. The employee clicks a Click-to-Call link or button in an application (e.g., CRM, email client)
2. The script initiates a call to the employee's extension
3. The extension forwards the call to the employee's mobile phone
4. When the employee answers, the PBX connects them to the outgoing number
5. All calls are billed according to the PBX dial plan, not the employee's mobile plan

**Note:** By default only private client IPs are allowed. Internet access requires VPN and/or IP whitelist changes plus API authentication.

### Integration Example: Outlook Plugin

This setup has been successfully integrated into a custom Outlook plugin (not part of this project), allowing remote employees to initiate calls directly from telephone numbers listed in emails. When the user clicks a number:

1. The Outlook plugin calls the Click-to-Call API
2. The PBX initiates the call to the employee's extension
3. The call is forwarded to their mobile
4. Upon answering, the call is connected to the clicked number

This eliminates the need for manual dialing and ensures all calls appear to come from the company's phone system.

## API Documentation

The Click-to-Call script provides a simple HTTP API that returns JSON responses. This makes it easy to integrate with other applications.

### Endpoint

```
GET /click.php
```

### Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `exten` | Yes | The extension that will ring first. |
| `number` | Yes | The phone number to call after the extension answers. |

### Response Format

The API always returns a JSON object with the following fields:

| Field | Type | Description |
|-------|------|-------------|
| `Success` | boolean | Indicates whether the call was initiated successfully. |
| `ValidInput` | boolean | Indicates whether the input parameters were valid. |
| `Description` | string | A human-readable description of the result or error. |
| `Technology` | string | The technology used for the extension (PJSIP or SIP). Only present on successful calls. |
| `OriginateResponse` | string | The raw response from the Asterisk Manager Interface. Only present on successful calls. |

### Response Examples

#### Successful Response

```json
{
  "Success": true,
  "ValidInput": true,
  "Description": "Extension 123 is calling 0612345678.",
  "Technology": "PJSIP",
  "OriginateResponse": "Response: Success\r\nMessage: Originate successfully queued\r\n\r\n"
}
```

#### Invalid Number Format

```json
{
  "Success": false,
  "ValidInput": false,
  "Description": "Invalid number format: abc123",
  "Technology": "",
  "OriginateResponse": ""
}
```

#### Invalid Extension Format

```json
{
  "Success": false,
  "ValidInput": false,
  "Description": "Invalid extension format: abc",
  "Technology": "",
  "OriginateResponse": ""
}
```

#### Unauthorized IP Address

```json
{
  "Success": false,
  "ValidInput": false,
  "Description": "Unauthorized IP address: 203.0.113.1",
  "Technology": "",
  "OriginateResponse": ""
}
```

#### Authentication Failure

```json
{
  "Success": false,
  "ValidInput": false,
  "Description": "Authentication failed",
  "Technology": "",
  "OriginateResponse": ""
}
```

#### Technology Retrieval Failure

```json
{
  "Success": false,
  "ValidInput": false,
  "Description": "Failed to retrieve technology for extension: 123",
  "Technology": "",
  "OriginateResponse": ""
}
```

#### Call Initiation Failure

```json
{
  "Success": false,
  "ValidInput": false,
  "Description": "Call initiation failed",
  "Technology": "PJSIP",
  "OriginateResponse": "Response: Error\r\nMessage: Channel PJSIP/123 not available\r\n\r\n"
}
```

## Security Considerations

The Click-to-Call script includes several security features to protect your PBX system:

### IP Address Validation

The script checks if the request is coming from a local IP address (private range). This prevents unauthorized access from the public internet. The relevant code is:

```php
if (filter_var($_SERVER["REMOTE_ADDR"], FILTER_VALIDATE_IP, FILTER_FLAG_NO_PRIV_RANGE | FILTER_FLAG_NO_RES_RANGE)) {
    $result = setError($result, "Unauthorized IP address: %s", $_SERVER["REMOTE_ADDR"]);
}
```

If you need to allow access from specific public IP addresses, you'll need to modify this check.

### Input Validation

All input parameters are validated to prevent injection attacks:

- Extension must contain only digits
- Number must contain only digits and optionally a leading plus sign

## Troubleshooting

If you encounter issues with the Click-to-Call script, here are some common problems and solutions:

### Extension Does Not Ring

- **Check Extension Registration**: Verify the extension is correctly registered and reachable.
  ```bash
  asterisk -rx "pjsip show endpoint <extension>"  # For PJSIP
  asterisk -rx "sip show peer <extension>"        # For CHAN_SIP
  ```

- **Check Dialplan Context**: Ensure the context (`from-internal` by default) is correctly configured.
  ```bash
  asterisk -rx "dialplan show from-internal"
  ```

- **Inspect Firewall and NAT Settings**: Ensure proper connectivity.
  ```bash
  iptables -L -n
  ```

### Authentication Failures

- **Verify AMI Credentials**: Check that the username and password in the script match those in `/etc/asterisk/manager.conf`.
  ```bash
  grep -A 10 "\[admin\]" /etc/asterisk/manager.conf
  ```

- **Check AMI Permissions**: Ensure the AMI user has the necessary permissions.
  ```
  read = system,call,log,verbose,command,agent,user,config,dtmf,reporting,cdr,dialplan,originate
  write = system,call,log,verbose,command,agent,user,config,dtmf,reporting,cdr,dialplan,originate
  ```

### Technology Detection Issues

- **Check DEVICE Database**: Verify the extension's technology is correctly stored in the Asterisk database.
  ```bash
  asterisk -rx "database show DEVICE"
  ```

### Debugging with Asterisk CLI

For detailed debugging, use the Asterisk CLI with increased verbosity:

```bash
asterisk -rvvvvv
```

Then watch for events related to your extension and calls:

```
core set verbose 5
```

### Checking Logs

Review Asterisk logs for more information:

```bash
tail -f /var/log/asterisk/full
```

## Version History

### v1.0.1 (2026-11-23)

- Renamed main script to `click.php`
- Full Russian and English documentation in README
- Added CHANGELOG.md; versioning scheme 1.0.x

### v1.0.0 (2023-04-01)

- Initial release
- Support for PJSIP and CHAN_SIP
- JSON response format
- Input validation
- IP address security check

## License

This project is licensed under the MIT License - see the LICENSE file for details.

## Credits

This project has been partly developed in time donated by [Contour - Sheet Metal Supplier](https://www.contour.eu/en/).

Dit project is deels ontwikkeld in de tijd van [Contour - Plaatwerkleverancier](https://www.contour.eu/plaatwerk/).

Adapted from [Alisson Pelizaro's Asterisk Click-to-Call](https://github.com/alissonpelizaro/Asterisk-Click-to-Call) script.