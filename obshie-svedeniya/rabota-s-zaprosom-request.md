# Работа с запросом \(Request\)

Для работы с данными HTTP запроса в JohnCMS используется класс **\Johncms\System\Http\Request**. Он позволяет получить доступ к таким суперглобальным переменным как: **$\_POST, $\_GET, $\_COOKIE, $\_FILES, $\_SERVER**. Это позволяет упростить получение значений по умолчанию, и фильтрацию данных, пришедших от пользователя. Давайте посмотрим на примеры.

Для начала необходимо получить объект класса Request.

```php
/** @var \Johncms\System\Http\Request $request */
$request = di(\Johncms\System\Http\Request::class);
```

Строка /\*\* @var \Johncms\System\Http\Request $request \*/ не обязательна и служит лишь для работы автодополнения в IDE если вы конечно используете IDE.

## Получение данных из $\_GET

Предположим, что пользователь открыл страницу http://domain.com/?user\_id=123 и нам нужно получить идентификатор пользователя 123. Сделать это можно следующим образом:

```php
$user = $request->getQuery('user_id', 0, FILTER_VALIDATE_INT);
```

Разберем что же тут происходит. Метод **getQuery** пытается получить **user\_id** из суперглобального массива **$\_GET**.  
Первым параметром принимает название параметра запроса, вторым параметром можно передать стандартное значение, а третим параметром передается фильтр, с помощью которого будет обработано значение. Вы можете ознакомиться со списком фильтров в официальной документации по этой ссылке: [https://www.php.net/manual/ru/filter.filters.php](https://www.php.net/manual/ru/filter.filters.php)

Коротко что делает строка из примера: Пытается получить параметр GET запроса **user\_id**, если его нет, то возвращает 0, если есть, то очищает и возвращает число. Если передано не число, то вернет значение по умолчанию, то есть 0.

## Получение данных из $\_POST

Предположим, что отправлена форма, которая содержит **user\_id** и **name**. Форма отправлена методом POST.

```php
$user = $request->getPost('user_id', 0, FILTER_VALIDATE_INT);
$name = $request->getPost('name', '', FILTER_SANITIZE_STRING);
```

Эти примеры работают так же как и предыдущий. Во втором примере от пользователя ожидается строка, а фильтр **FILTER\_SANITIZE\_STRING** удаляет из нее теги, и при необходимости удаляет или кодирует специальные символы.

## Получение данных из $\_COOKIE

```php
$name = $request->getCookie('name', '', FILTER_SANITIZE_STRING);
```

В этом примере как видите всё так же просто как и в предыдущих. Просто поменялось название метода, а принцип работы такой же.

## Получение данных из $\_SERVER

```php
$user_agent = $request->getServer('HTTP_USER_AGENT', '', FILTER_SANITIZE_STRING);
```

Этот пример работает так же как и остальные. Получает **HTTP\_USER\_AGENT** из суперглобальной переменной **$\_SERVER**.

## Получение данных из $\_FILES

```php
$files = $request->getUploadedFiles();
```

Этот код вернет массив файлов, в котором каждый элемент будет представлен объектом класса GuzzleHttp\Psr7\UploadedFile. Если вы уже работали с выгрузкой файлов в php, то наверное знаете, что множественные файлы в массиве $\_FILES описываются примерно так:

```php
array(
    'files' => array(
        'name' => array(
            0 => 'file0.txt',
            1 => 'file1.html',
        ),
        'type' => array(
            0 => 'text/plain',
            1 => 'text/html',
        ),
        /* etc. */
    ),
)
```

для работы с этим стандартными средствами вам необходимо позаботиться о сборе всех данных в нормальную структуру. Если вы используете метод **getUploadedFiles**, то эта задача уже решена для вас и массив файлов будет уже в нормальной структуре:

```php
array(
    'files' => array(
        0 => array(
            'name' => 'file0.txt',
            'type' => 'text/plain',
            /* etc. */
        ),
        1 => array(
            'name' => 'file1.html',
            'type' => 'text/html',
            /* etc. */
        ),
    ),
)
```

Давайте рассмотрим пример сохранения файлов

Допустим, у нас есть такая форма, которая принимает 1 обычный файл и поле с возможностью выбирать несколько файлов.

```markup
<form action="" method="post" enctype="multipart/form-data">
    <input type="file" name="file">
    <input type="file" name="multiple_files[]" multiple>
    <button type="submit">Отправить</button>
</form>
```

Пример сохранения файлов будет выглядеть так:

```php
$files = $request->getUploadedFiles();

// Сохраняем файл из обычного поля
if (! empty($files['file'])) {
    /** @var  $attached_file \Psr\Http\Message\UploadedFileInterface */
    $attached_file = $files['file'];
    try {
        $attached_file->moveTo(UPLOAD_PATH . '/tmp/' . $attached_file->getClientFilename());
        echo 'Файл успешно сохранен';
    } catch (\Exception $exception) {
        echo 'Ошибка сохранения файла: ' . $exception->getMessage();
    }
}

// Сохраняем файлы из множественного поля
if (! empty($files['multiple_files'])) {
    /** @var  $multiple_files \Psr\Http\Message\UploadedFileInterface[] */
    $multiple_files = $files['multiple_files'];
    foreach ($multiple_files as $multiple_file) {
        try {
            $multiple_file->moveTo(UPLOAD_PATH . '/tmp/' . $multiple_file->getClientFilename());
            echo 'Файл успешно сохранен';
        } catch (\Exception $exception) {
            echo 'Ошибка сохранения файла: ' . $exception->getMessage();
        }
    }
}
```

В результате отправки формы с файлами, все файлы будут сохранены в папке upload/tmp c оригинальными названиями, с которыми отправил клиент.

{% hint style="danger" %}
Обратите внимание, что в примере рассмотрен простой вариант сохранения файлов без каких либо проверок допустимых типов файлов.
{% endhint %}

