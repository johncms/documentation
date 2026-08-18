---
description: Как уменьшать, конвертировать и превьюить загруженные картинки
---

# Обработка изображений

Аватар, фото профиля, скриншот файла, снимок в альбоме — всё это загружает посетитель, и хранить такое в исходном виде нельзя: с телефона придёт снимок на 12 мегапикселей, который положит вёрстку списка и съест место на диске. Уменьшением, обрезкой и перекодированием занимается **процессор изображений**.

## Главное правило

**Обращайтесь к `Johncms\Image\ImageProcessorInterface`, а не к библиотеке напрямую.**

За интерфейсом стоит [Intervention Image](https://image.intervention.io/), но её имя не должно встречаться нигде, кроме единственной реализации `InterventionImageProcessor`. Так переход на новую мажорную версию библиотеки — правка одного файла, а не всех мест, где сохраняется картинка.

{% hint style="info" %}
Это тот же приём, что и у [санитайзера HTML](html-sanitizer.md): вызывающий код говорит, **что** ему нужно получить, а чем это сделано — деталь реализации.
{% endhint %}

## Как пользоваться

Внедрите интерфейс через конструктор:

```php
<?php

declare(strict_types=1);

namespace Johncms\Modules\MyModule\Application\UseCases;

use Johncms\Http\UploadedFileDTO;
use Johncms\Image\ImageProcessingException;
use Johncms\Image\ImageProcessorInterface;

final readonly class UploadCoverUseCase
{
    private const int COVER_WIDTH = 800;
    private const int COVER_HEIGHT = 600;

    public function __construct(
        private ImageProcessorInterface $imageProcessor,
    ) {
    }

    public function execute(int $id, UploadedFileDTO $file): void
    {
        try {
            $this->imageProcessor->saveScaledDown(
                $file->tmpPath,
                UPLOAD_PATH . 'covers/' . $id . '.jpg',
                self::COVER_WIDTH,
                self::COVER_HEIGHT
            );
        } catch (ImageProcessingException $exception) {
            throw new CoverUploadException($exception->getMessage());
        }
    }
}
```

На входе и на выходе — **пути к файлам**: загрузка и так лежит на диске (`UploadedFileDTO::$tmpPath`), результат тоже нужен на диске.

### Доступные операции

| Метод | Что делает |
|---|---|
| `saveScaledDown($source, $target, $width, $height, $quality)` | Вписывает картинку в заданные границы с сохранением пропорций. Картинку меньше границ **не увеличивает** — оставляет как есть |
| `saveBlurredThumbnail($source, $target, $width, $height, $quality)` | Миниатюра ровно заданного размера: картинка обрезается под размер и размывается как подложка, а сверху по центру кладётся уменьшенная копия |
| `saveConverted($source, $target, $quality)` | Та же картинка в исходном размере, перекодированная в другой формат |

Стороны в `saveScaledDown()` необязательные: `null` означает «не ограничивать». Чтобы уменьшить только по ширине, передайте одну её:

```php
$this->imageProcessor->saveScaledDown($source, $target, 240);
```

{% hint style="success" %}
`saveBlurredThumbnail()` нужен там, где миниатюры стоят сеткой одинаковых плиток: размытая подложка позволяет положить в плитку картинку любых пропорций, не оставляя пустых полей по краям. Так сделаны миниатюры альбома и превью скриншотов в загрузках.
{% endhint %}

### Формат результата

**Формат берётся из расширения целевого файла.** Сохранение в `avatar.png` даст PNG, в `photo.jpg` — JPEG, независимо от того, что загрузил посетитель.

{% hint style="danger" %}
Если файл нужно сохранить под фиксированным расширением, не копируйте его функцией `copy()` — картинку нужно перекодировать, иначе в файле `.png` окажутся байты JPEG:

```php
// Неправильно: имя .png, содержимое JPEG
copy($source, UPLOAD_PATH . 'library/images/orig/' . $id . '.png');

// Правильно
$this->imageProcessor->saveConverted($source, UPLOAD_PATH . 'library/images/orig/' . $id . '.png');
```
{% endhint %}

## Обработка ошибок

Всё, с чем процессор не справился — битый файл, неподдерживаемый формат, недоступная для записи папка, — приходит одним исключением `Johncms\Image\ImageProcessingException`.

```php
try {
    $this->imageProcessor->saveScaledDown($source, $target, 400, 300);
} catch (ImageProcessingException $exception) {
    // сообщить пользователю, что картинку не удалось загрузить
}
```

{% hint style="warning" %}
Не ловите `\Exception` вокруг работы с картинками. Такой блок вместе с неудачной загрузкой проглотит и настоящую ошибку в коде, а посетитель увидит «не удалось загрузить изображение» там, где на самом деле сломался модуль.
{% endhint %}

## Превью

Превью — картинка, которую показывают вместо оригинала в списке. Генерировать её на каждый показ дорого: страница с двадцатью вложениями означала бы двадцать раз декодировать, уменьшить и размыть JPEG. Поэтому превью считается один раз и складывается на диск.

Этим занимается `Johncms\Image\ThumbnailGenerator`. Он возвращает путь к готовому файлу, а пересчитывает его, только если файла нет или оригинал новее:

```php
// Плитка ровно 220×300 с размытой подложкой, всегда JPEG
$preview = $this->thumbnails->blurredBackdrop($path, 220, 300);

// Уменьшенная копия, формат исходника сохраняется
$preview = $this->thumbnails->scaledDown($path, 100, 100);
```

Кэш лежит в `data/cache/thumbnails` — **вне папки сайта**, и это сделано намеренно: превью отдаёт контроллер, а значит право посетителя видеть эту картинку проверяется, чего каталог, раздаваемый веб-сервером напрямую, обеспечить не может.

Чистится вместе с остальным кэшем, отдельно ничего настраивать не нужно:

```bash
php system/bin/console cache:clear
```

### Отдача превью

Готовый файл отдаётся классом `Johncms\Http\CachedImageResponse`:

```php
<?php

declare(strict_types=1);

namespace Johncms\Modules\MyModule\Application\Controllers;

use Johncms\Http\CachedImageResponse;
use Johncms\Http\Request;
use Johncms\Image\ImageProcessingException;
use Johncms\Image\ThumbnailGenerator;
use Symfony\Component\HttpFoundation\Response;

final readonly class CoverPreviewController
{
    public function __construct(
        private CoverRepositoryInterface $covers,
        private ThumbnailGenerator $thumbnails,
    ) {
    }

    public function __invoke(Request $request, int $id): Response
    {
        $cover = $this->covers->findById($id);
        if ($cover === null) {
            return new Response('', Response::HTTP_NOT_FOUND);
        }

        try {
            $preview = $this->thumbnails->scaledDown(UPLOAD_PATH . 'covers/' . $cover->file, 220, 300);
        } catch (ImageProcessingException) {
            return new Response('', Response::HTTP_NOT_FOUND);
        }

        $response = new CachedImageResponse($preview);
        // Повторный запрос после истечения срока кэша получит 304 вместо картинки
        $response->isNotModified($request);

        return $response;
    }
}
```

`CachedImageResponse` сам проставляет `Content-Type`, `Content-Length`, `Last-Modified` и `Cache-Control: max-age=..., private`. Собирать `BinaryFileResponse` вручную не нужно: ядро намеренно не вызывает `Response::prepare()`, поэтому обычный файловый ответ ушёл бы с типом `text/html`.

{% hint style="info" %}
Кэш помечен `private`, а не `public`: раздающий прокси перед сайтом иначе отдал бы сохранённую копию всем подряд, включая тех, кому картинка не предназначена. Браузеру самого посетителя это не мешает — именно он и делает повторные запросы на странице, полной миниатюр.
{% endhint %}

### Никогда не берите путь из запроса

Адресуйте картинку **по идентификатору записи, которой она принадлежит**, а путь стройте в контроллере:

```php
// Правильно
$r->get('/my-module/preview/{id:number}', CoverPreviewController::class)->name('my_module.preview');
```

Ссылка вида `preview.php?img=/upload/...` — приглашение к обходу каталогов: имя с `../` внутри уводит чтение куда угодно по диску. Если имя файла в адресе всё же неизбежно (например, конкретный скриншот внутри папки), пропустите его через `basename()`, ограничьте маршрут регулярным выражением и **дополнительно** проверьте, что `realpath()` результата остался внутри нужной папки:

```php
$r->get('/my-module/preview/{id:number}/{name}', [CoverPreviewController::class, 'screen'])
    ->requirements(['name' => '[A-Za-z0-9_.\-]+']);
```

## Драйверы и настройки

Обработка идёт через расширение **Imagick**, если оно установлено, иначе через **GD**. Выбор делается автоматически, настраивать ничего не нужно.

Правила декодирования заданы в одном месте — методе `manager()` класса `InterventionImageProcessor`:

| Настройка | Значение | Зачем |
|---|---|---|
| `autoOrientation` | включена | Снимок с телефона хранит поворот в EXIF, а не в пикселях. Без этого фото сохранялось бы лежащим на боку |
| `decodeAnimation` | выключена | CMS хранит один кадр. Раскладывать анимированный GIF покадрово — значит обработать каждый кадр и записать первый |
| `strip` | включена | Метаданные снимка содержат GPS-координаты места съёмки, а сохранённый файл публичный |

{% hint style="warning" %}
Автоповороту нужно расширение `ext-exif`. В поставляемом Docker-образе оно есть; если на хостинге его нет, библиотека просто не станет поворачивать картинку — ошибки не будет.
{% endhint %}

## Если нужной операции нет

Добавьте метод в `ImageProcessorInterface` и реализуйте его в `InterventionImageProcessor`. Название метода описывает **задачу**, а не действие библиотеки: `saveBlurredThumbnail()`, а не `crop()` плюс `blur()` плюс `insert()`.

{% hint style="danger" %}
Не превращайте интерфейс в копию API библиотеки. Набор из `resize()`, `crop()`, `blur()` не даёт ничего: он повторяет то, что оборачивает, и возвращает знание о библиотеке во все вызывающие места — ровно та зависимость, ради избавления от которой интерфейс и существует.
{% endhint %}
