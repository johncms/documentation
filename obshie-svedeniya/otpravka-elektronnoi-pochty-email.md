# Отправка электронной почты \(email\)

В JohnCMS для отправки электронной почты используется библиотека [laminas-mail ](https://docs.laminas.dev/laminas-mail/)  
Она позволяет обобщить отправку сообщений и легко переключать драйверы через которые будет отправляться письмо. Благодаря этому вы сможете выбрать наиболее подходящий вам метод отправки в зависимости от возможностей вашего хостинга и наличия его ip в спам фильтрах.

### Драйверы и настройка

На данный момент поддерживаются следующие драйверы: **Sendmail, SMTP, File.** Этих драйверов обычно более чем достаточно большинству проектов.

Драйвер по умолчанию и настройки драйвера указываются в конфигурационном файле **config/autoload/mail.global.php**. По умолчанию установлен sendmail, но вы можете сменить драйвер на smtp или file. Примеры настроек есть в указанном файле. Вы можете просто их переопределить. Как это сделать, а так же про работу с конфигурационными файлами рекомендуем прочитать здесь: [Конфигурационные файлы.](https://johncms.com/documentation/configs/)

### Отправка сообщений

Отправка email достаточно затратная операция. Для решения этой проблемы отправку email можно переложить на сервер. Для этого в JohnCMS реализована очередь сообщений. Чтобы отправить письмо, необходимо просто добавить его в очередь.

**Рассмотрим пример добавления письма в очередь:**

```php
(new \Johncms\Mail\EmailMessage())->create(
    [
        'locale'   => 'ru',
        'template' => 'system::mail/templates/registration',
        'fields'   => [
            'email_to'        => 'user@example.com',
            'name_to'         => 'Имя Пользователя',
            'subject'         => 'Регистрация на сайте',
            'user_name'       => 'UserName',
            'user_login'      => 'UserLogin',
            'link_to_confirm' => 'https://johncms.com',
        ],
    ]
);
```

Что делает этот код?  
Он добавляет запись в таблицу **email\_messages**. А дальше система проверяет наличие не отправленных писем в очереди и отправляет их.

#### Какие поля необходимы?

* **priority** - Приоритет отправки сообщения. Чем меньше, тем выше. \(**не обязательно**\)
* **locale** - Поле обязательно и содержит код языка, на котором будет отправлено сообщение.
* **template** - содержит шаблон, который будет использоваться для формирования письма.
* **fields** - содержит массив полей, которые будут доступны в шаблоне, а так же будут использоваться для отправки:
  * **email\_to** - E-mail адрес получателя сообщения \(**обязательное поле**\)
  * **name\_to** - Имя получателя, которое будет отображаться в почтовом клиенте. \(**не обязательно**\)
  * **subject** - Тема сообщения. \(**не обязательно, но рекомендуется**\)
  * Прочие поля доступны только в шаблоне, не требуются для работы драйвера и могут отсутствовать.

Отправка почтовых сообщений по умолчанию выполняется на хитах. Это значит, что для отправки письма какой либо пользователь должен зайти на сайт. В момент на сайт, выполняется проверка наличия неотправленных сообщений и если таковые находятся, выполняется отправка. Этот вариант не всегда подходит, особенно если сообщений отправляется много. По этому рекомендуется перевести отправку сообщений на cron.

### Перевод отправки Email на CRON

Для того, чтобы избежать подвисания страницы для пользователей в JohnCMS реализована отправка сообщений с помощью планировщика cron. Если ваш хостинг поддерживает cron, то рекомендуем перевести отправку сообщений на него. Для этого откройте файл: **config/constants.php**  
Найдите строчку:

```php
const USE_CRON = false;
```

И замените **false** на **true**.

Далее необходимо добавить задачу в cron:

**php system/cron.php**

Периодичность выполнения установить раз в 1 минуту.  
Обратите внимание, что может потребоваться указать полный путь к файлу от корня. Посмотреть его можно в **phpinfo\(\)**, параметр **DOCUMENT\_ROOT** или вывести так:   
**echo $\_SERVER\['DOCUMENT\_ROOT'\];**  
Более подробно про то как добавить задачу, вы можете уточнить у вашего хостинг провайдера.
