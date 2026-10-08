# Настройка почтового сервера support@example.com на Windows Server 2019 с hMailServer и ASP.NET Core 10

> Полное руководство: от установки зависимостей и hMailServer до интеграции с C# / ASP.NET Core 10.

## Содержание

- [Предварительные требования](#предварительные-требования)
- [Шаг 1. Установка зависимостей (.NET Framework)](#шаг-1-установка-зависимостей-net-framework)
- [Шаг 2. Установка hMailServer](#шаг-2-установка-hmailserver)
- [Шаг 3. Настройка домена и почтового ящика](#шаг-3-настройка-домена-и-почтового-ящика)
- [Шаг 4. Настройка DNS-записей](#шаг-4-настройка-dns-записей)
- [Шаг 5. Защита от спама](#шаг-5-защита-от-спама)
- [Шаг 6. Интеграция с ASP.NET Core 10](#шаг-6-интеграция-с-aspnet-core-10)
- [Шаг 7. Проверка и тестирование](#шаг-7-проверка-и-тестирование)
- [Приложение. Полезные ссылки](#приложение-полезные-ссылки)

---

## Предварительные требования

Перед установкой убедитесь, что выполнены следующие условия:

- **Windows Server 2019** (Standard или Datacenter).
- **.NET Framework 4.8** — требуется установщиком hMailServer[reference:0].
- **.NET Framework 3.5** — требуется для работы административной панели hMailServer Administrator[reference:1].
- **Статический публичный IP-адрес** — необходим для корректной работы MX-записей и PTR-записи.
- **Доступ к DNS-панели домена** `example.com` (например, Cloudflare, REG.RU и т.п.).
- **Открытые порты** на брандмауэре:

| Порт | Протокол | Назначение |
|------|----------|------------|
| 25   | TCP      | SMTP (приём входящей почты) |
| 587  | TCP      | SMTP Submission (отправка с аутентификацией) |
| 465  | TCP      | SMTPS (устаревший, но иногда используется) |
| 143  | TCP      | IMAP |
| 993  | TCP      | IMAPS |
| 110  | TCP      | POP3 |
| 995  | TCP      | POP3S |

> ⚠️ **Важно:** Многие облачные провайдеры блокируют порт 25 по умолчанию. Уточните у хостера возможность его разблокировки, иначе исходящие письма не будут доставляться.

---

## Шаг 1. Установка зависимостей (.NET Framework)

hMailServer зависит от **двух** версий .NET Framework, и обе нужно подготовить **до** установки hMailServer.

### 1.1. Установка .NET Framework 4.8

В Windows Server 2019 .NET Framework 4.8 обычно уже входит в состав системы. Проверить версию можно командой в PowerShell:

```powershell
Get-ItemProperty "HKLM:\SOFTWARE\Microsoft\NET Framework Setup\NDP\v4\Full" | Select-Object Version, Release
```

- Если `Release` ≥ **528040** — .NET 4.8 уже установлен, дополнительных действий не требуется.
- Если .NET 4.8 отсутствует, скачайте установщик с [официальной страницы Microsoft](https://dotnet.microsoft.com/download/dotnet-framework/net48) и установите его.

### 1.2. Включение .NET Framework 3.5

.NET Framework 3.5 **отключён по умолчанию** в Windows Server 2019. Его нужно включить через «Add Roles and Features Wizard»[reference:2].

**Способ 1: через графический интерфейс (Server Manager)**

1. Откройте **Start Menu** → найдите **Add Roles and Features Wizard** и запустите его.
2. Пройдите по шагам мастера до раздела **Features** (Функции).
3. В списке функций найдите и установите галочку **.NET Framework 3.5 Features** (включает .NET 2.0 и 3.0)[reference:3].
4. Нажмите **Install** и дождитесь завершения установки.

**Способ 2: через PowerShell (офлайн-установка, если нет доступа к Windows Update)**

Если сервер не имеет доступа к интернету или Windows Update, используйте установочный ISO-образ Windows Server 2019:

```powershell
# Подключите ISO-образ Windows Server 2019 и укажите букву диска (например, D:)
# Затем выполните команду:
Install-WindowsFeature NET-Framework-Core -Source D:\sources\sxs
```

> 💡 **Примечание:** В современных версиях Windows Server полезная нагрузка компонента .NET 3.5 удалена из базового образа ОС, поэтому при офлайн-установке обязательно указывать папку `sources\sxs` с установочного носителя[reference:4].

**Проверка:**

```powershell
Get-WindowsFeature NET-Framework-Core
# Параметр Install State должен быть "Installed"
```

---

## Шаг 2. Установка hMailServer

1. **Скачайте** последнюю стабильную версию hMailServer с [официального сайта](https://www.hmailserver.com/).  
   Актуальная версия на момент написания — **5.6.8-B2425**.

2. **Запустите установщик** и выберите **Full Installation**.

3. На этапе выбора базы данных:
   - Для небольшой нагрузки (до 50 ящиков) подойдёт **встроенный движок** (built-in database engine).
   - Для продакшена рекомендуется **MySQL 5.6+ / MariaDB** с кодировкой `utf8mb4`.

4. **Задайте пароль администратора** (не менее 12 символов) и сохраните его в надёжном месте.

5. Установите галочку **«Start hMailServer service after installation»**, чтобы служба запускалась автоматически.

6. После установки откройте **hMailServer Administrator** и подключитесь, введя пароль администратора.

> ⚠️ Если при запуске hMailServer Administrator появляется ошибка, проверьте, что .NET Framework 3.5 включён (см. Шаг 1.2).

---

## Шаг 3. Настройка домена и почтового ящика

### 3.1. Добавление домена

1. В дереве слева выберите **Domains → Add**.
2. Введите доменное имя: `example.com`.
3. Нажмите **Save**.

### 3.2. Создание ящика `support@example.com`

1. Разверните созданный домен и выберите **Accounts → Add**.
2. Укажите:
   - **Address:** `support@example.com`
   - **Password:** надёжный пароль (например, `YOUR_STRONG_PASSWORD`)
3. Нажмите **Save**.

### 3.3. Включение SMTP и настройка доставки

1. Перейдите в **Settings → Protocols → SMTP**.
2. Убедитесь, что SMTP включён.
3. В разделе **Delivery of Email**:
   - **Local host name:** `mail.example.com`
   - **SMTP Relayer:** оставьте пустым, если порт 25 открыт. Если провайдер блокирует порт 25, укажите релей вашего хостера.
   - **Maximum connections:** 10 (при необходимости увеличьте).
4. Включите **STARTTLS** для шифрования передачи.

### 3.4. Настройка диапазонов IP

1. Перейдите в **Settings → Advanced → IP Ranges**.
2. Для диапазона **Internet** убедитесь, что включена опция **Allow deliveries from external to local accounts** — это разрешит приём входящих писем из интернета.

### 3.5. Установка SSL-сертификата

Для безопасной работы IMAP/SMTP рекомендуется установить SSL-сертификат (например, Let's Encrypt).

1. Получите сертификат для `mail.example.com` с помощью [win-acme](https://www.win-acme.com/) или другого клиента.
2. Установите сертификат в хранилище сертификатов Windows.
3. В hMailServer Administrator перейдите в **Settings → Advanced → SSL certificates**.
4. Добавьте сертификат, выбрав его из хранилища.
5. Привяжите сертификат к портам:
   - **Settings → Advanced → TCP/IP ports** → для портов 993 (IMAPS), 995 (POP3S), 465 (SMTPS) выберите созданный сертификат.

---

## Шаг 4. Настройка DNS-записей

Все записи добавляются в панели управления DNS вашего домена `example.com`. Ниже — минимальный набор для корректной работы почты.

| Тип записи | Имя (Host) | Значение (Value) | Приоритет | Назначение |
|---|---|---|---|---|
| **A** | `mail` | `YOUR_PUBLIC_IP` | — | Привязка почтового сервера к IP |
| **MX** | `@` | `mail.example.com` | 10 | Указывает, какой сервер принимает почту |
| **TXT (SPF)** | `@` | `v=spf1 mx ip4:YOUR_PUBLIC_IP -all` | — | Разрешает отправку только с вашего сервера |
| **TXT (DKIM)** | `default._domainkey` | `v=DKIM1; k=rsa; p=YOUR_PUBLIC_KEY` | — | Криптографическая подпись писем |
| **TXT (DMARC)** | `_dmarc` | `v=DMARC1; p=none; rua=mailto:support@example.com` | — | Политика обработки поддельных писем |

> 🔑 **DKIM-ключ:** Сгенерируйте пару ключей (приватный + публичный) с помощью бесплатного генератора, например [SocketLabs DKIM Generator](https://tools.socketlabs.com/dkim/generator), или через OpenSSL:
>
> ```bash
> openssl genrsa -out private.key 2048
> openssl rsa -in private.key -pubout -out public.key
> ```
>
> Приватный ключ загрузите в hMailServer (раздел **Settings → Advanced → DKIM Signing**), публичный — в DNS-запись.

> 🌐 **PTR-запись (Reverse DNS):** Обратитесь к вашему хостинг-провайдеру с запросом на настройку PTR-записи для вашего IP-адреса, указав значение `mail.example.com`. Без неё многие почтовые сервисы (Gmail, Outlook) будут отклонять ваши письма как спам.

---

## Шаг 5. Защита от спама

hMailServer имеет встроенные механизмы антиспама. Перейдите в **Settings → Anti-spam**.

### 5.1. Основные тесты

| Параметр | Рекомендуемое значение | Описание |
|---|---|---|
| **Spam mark threshold** | 5 | Сообщения с оценкой ≥ 5 помечаются как спам |
| **Spam delete threshold** | 15 | Сообщения с оценкой ≥ 15 удаляются |
| **Use SPF** | ✅ Включено | Проверка соответствия IP отправителя SPF-записи |
| **Check host in HELO** | ✅ Включено | Проверка соответствия HELO-хоста IP клиента |
| **Check sender has MX records** | ✅ Включено | Отклонение писем от доменов без MX-записей |
| **Verify DKIM-Signature** | ✅ Включено | Проверка подписи DKIM входящих писем |

### 5.2. DNS-блэклисты (RBL)

1. Перейдите в **Settings → Anti-spam → DNS Blacklists**.
2. Добавьте известные блэклисты:
   - `zen.spamhaus.org`
   - `bl.spamcop.net`
   - `b.barracudacentral.org`
3. Включите опцию **«Check sender's IP address against DNS blacklists»**.

### 5.3. Грейлистинг (Greylisting)

Рекомендуется включить в **Settings → Anti-spam → Greylisting**. Этот механизм временно отклоняет первое письмо от неизвестного отправителя, заставляя его повторить попытку через некоторое время. Большинство спам-ботов этого не делают.

### 5.4. Тарпиттинг (Tarpitting)

Опция **Tarpitting** позволяет замедлить ответы сервера при большом количестве получателей в одном SMTP-сеансе, что отпугивает спамеров. Используйте с осторожностью, так как может замедлить и легитимную рассылку.

> 💡 **Интеграция со SpamAssassin:** Для продвинутой фильтрации можно установить SpamAssassin отдельно и подключить его в hMailServer (раздел **Anti-spam → SpamAssassin**, хост `localhost`, порт 783).

---

## Шаг 6. Интеграция с ASP.NET Core 10

Для работы с почтой в .NET рекомендуется использовать библиотеку **MailKit** вместо устаревшего `System.Net.Mail.SmtpClient`.

### 6.1. Установка пакетов

Выполните в Package Manager Console или через CLI:

```bash
dotnet add package MailKit
dotnet add package MimeKit
```

### 6.2. Конфигурация приложения

Добавьте настройки в `appsettings.json`:

```json
{
  "Email": {
    "SmtpHost": "mail.example.com",
    "SmtpPort": 587,
    "ImapHost": "mail.example.com",
    "ImapPort": 993,
    "Username": "support@example.com",
    "Password": "YOUR_STRONG_PASSWORD"
  }
}
```

> 🔒 В продакшене храните пароль в переменных окружения или в Azure Key Vault / User Secrets.

### 6.3. Отправка письма (SMTP)

Создайте сервис для отправки писем:

```csharp
using MailKit.Net.Smtp;
using MailKit.Security;
using MimeKit;

public class EmailSender
{
    private readonly IConfiguration _config;

    public EmailSender(IConfiguration config)
    {
        _config = config;
    }

    public async Task SendEmailAsync(string to, string subject, string body)
    {
        var message = new MimeMessage();
        message.From.Add(new MailboxAddress("Support", _config["Email:Username"]!));
        message.To.Add(MailboxAddress.Parse(to));
        message.Subject = subject;

        var bodyBuilder = new BodyBuilder { HtmlBody = body };
        message.Body = bodyBuilder.ToMessageBody();

        using var client = new SmtpClient();
        await client.ConnectAsync(
            _config["Email:SmtpHost"]!,
            int.Parse(_config["Email:SmtpPort"]!),
            SecureSocketOptions.StartTls);

        await client.AuthenticateAsync(
            _config["Email:Username"]!,
            _config["Email:Password"]!);

        await client.SendAsync(message);
        await client.DisconnectAsync(true);
    }
}
```

### 6.4. Получение писем (IMAP)

Для чтения входящих писем используйте `ImapClient` из MailKit:

```csharp
using MailKit.Net.Imap;
using MailKit.Search;
using MailKit;
using MimeKit;

public class EmailReceiver
{
    private readonly IConfiguration _config;

    public EmailReceiver(IConfiguration config)
    {
        _config = config;
    }

    public async Task<List<MimeMessage>> GetUnreadEmailsAsync()
    {
        var messages = new List<MimeMessage>();

        using var client = new ImapClient();
        await client.ConnectAsync(
            _config["Email:ImapHost"]!,
            int.Parse(_config["Email:ImapPort"]!),
            SecureSocketOptions.SslOnConnect);

        await client.AuthenticateAsync(
            _config["Email:Username"]!,
            _config["Email:Password"]!);

        var inbox = client.Inbox;
        await inbox.OpenAsync(FolderAccess.ReadOnly);

        var uids = await inbox.SearchAsync(SearchQuery.NotSeen);

        foreach (var uid in uids)
        {
            var message = await inbox.GetMessageAsync(uid);
            messages.Add(message);
        }

        await client.DisconnectAsync(true);
        return messages;
    }
}
```

### 6.5. Фоновая проверка новых писем

Для периодической проверки входящих можно использовать `BackgroundService`:

```csharp
public class EmailPollingService : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;
    private readonly ILogger<EmailPollingService> _logger;

    public EmailPollingService(
        IServiceScopeFactory scopeFactory,
        ILogger<EmailPollingService> logger)
    {
        _scopeFactory = scopeFactory;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            using var scope = _scopeFactory.CreateScope();
            var receiver = scope.ServiceProvider.GetRequiredService<EmailReceiver>();

            try
            {
                var emails = await receiver.GetUnreadEmailsAsync();
                foreach (var email in emails)
                {
                    _logger.LogInformation(
                        "Новое письмо от {From}: {Subject}",
                        email.From,
                        email.Subject);

                    // TODO: обработка письма (сохранение в БД, автоответ и т.д.)
                }
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Ошибка при получении почты");
            }

            await Task.Delay(TimeSpan.FromMinutes(2), stoppingToken);
        }
    }
}
```

### 6.6. Регистрация сервисов в `Program.cs`

```csharp
using var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddScoped<EmailSender>();
builder.Services.AddScoped<EmailReceiver>();
builder.Services.AddHostedService<EmailPollingService>();

var app = builder.Build();

app.MapControllers();
app.Run();
```

### 6.7. Пример контроллера для отправки

```csharp
[ApiController]
[Route("api/[controller]")]
public class MailController : ControllerBase
{
    private readonly EmailSender _emailSender;

    public MailController(EmailSender emailSender)
    {
        _emailSender = emailSender;
    }

    [HttpPost("send")]
    public async Task<IActionResult> Send([FromBody] SendMailRequest request)
    {
        await _emailSender.SendEmailAsync(request.To, request.Subject, request.Body);
        return Ok(new { message = "Письмо отправлено" });
    }
}

public record SendMailRequest(string To, string Subject, string Body);
```

---

## Шаг 7. Проверка и тестирование

### 7.1. Проверка DNS

```bash
nslookup -type=mx example.com
nslookup -type=txt example.com
nslookup -type=txt default._domainkey.example.com
nslookup -type=txt _dmarc.example.com
```

### 7.2. Проверка портов

Убедитесь, что порты открыты извне:

```bash
telnet mail.example.com 25
telnet mail.example.com 587
telnet mail.example.com 993
```

### 7.3. Проверка SSL

```bash
openssl s_client -connect mail.example.com:993
```

### 7.4. Тестовая отправка

Отправьте письмо на сервисы проверки:

- `check-auth@verifier.port25.com` — вернёт отчёт о SPF, DKIM, DMARC.
- [mail-tester.com](https://www.mail-tester.com/) — покажет оценку спама.
- [MXToolbox](https://mxtoolbox.com/) — проверка MX, SPF, DKIM, DMARC, PTR.

### 7.5. Частые проблемы

| Проблема | Решение |
|----------|---------|
| Письма не уходят | Проверьте, не блокирует ли провайдер порт 25. Используйте SMTP Relayer. |
| Письма попадают в спам | Настройте PTR-запись, SPF, DKIM, DMARC. Проверьте IP в блэклистах. |
| Не приходят входящие | Проверьте MX-запись, порт 25, настройки IP Ranges в hMailServer. |
| Ошибка аутентификации в C# | Убедитесь, что используете правильные порт и тип шифрования (`StartTls` для 587, `SslOnConnect` для 993). |
| DKIM не подписывает | Проверьте, что приватный ключ загружен и включена подпись для домена. |
| hMailServer Administrator не запускается | Включите .NET Framework 3.5 (см. Шаг 1.2)[reference:5]. |
| Установщик hMailServer требует .NET 4.x | Убедитесь, что .NET Framework 4.8 установлен (см. Шаг 1.1)[reference:6]. |

---

## Приложение. Полезные ссылки

- [hMailServer — официальный сайт](https://www.hmailserver.com/)
- [hMailServer — системные требования](https://www.hmailserver.com/documentation/latest/?page=system_requirements)
- [MailKit — документация](https://github.com/jstedfast/MailKit)
- [MimeKit — документация](https://github.com/jstedfast/MimeKit)
- [win-acme — Let's Encrypt для Windows](https://www.win-acme.com/)
- [SocketLabs DKIM Generator](https://tools.socketlabs.com/dkim/generator)
- [MXToolbox](https://mxtoolbox.com/)
- [mail-tester.com](https://www.mail-tester.com/)
- [Установка .NET Framework 3.5 в Windows Server 2019 (Microsoft Learn)](https://learn.microsoft.com/ru-ru/dotnet/framework/install/dotnet-35-windows-server)

---

> **Лицензия:** Данная инструкция предоставлена «как есть». Перед применением в продакшене протестируйте все шаги в изолированной среде.