---
description: Как уменьшать, обрезать и кэшировать превью загруженных картинок
---

# Обработка изображений

Аватар, фото профиля, скриншот файла, снимок в альбоме — всё это загружает посетитель, и хранить такое в исходном виде нельзя: с телефона придёт снимок на 12 мегапикселей, который положит вёрстку списка и съест место на диске. Уменьшением, обрезкой, перекодированием и водяными знаками занимается **сервис обработки изображений**.

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
use Johncms\Modules\MyModule\Application\Exceptions\CoverUploadException;

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
        $dir = UPLOAD_PATH . 'covers' . DS;
        if (! is_dir($dir) && ! mkdir($dir, 0777, true) && ! is_dir($dir)) {
            throw new CoverUploadException(__('An error occurred'));
        }

        try {
            $this->imageProcessor->saveScaledDown(
                $file->tmpPath,
                $dir . $id . '.jpg',
                self::COVER_WIDTH,
                self::COVER_HEIGHT
            );
        } catch (ImageProcessingException $exception) {
            throw new CoverUploadException($exception->getMessage());
        }
    }
}
```

Источник и цель задаются **путями к файлам**: загрузка и так лежит на диске (`UploadedFileDTO::$tmpPath`), результат тоже нужен на диске. Методы ничего не возвращают — они пишут файл.

### Общие правила

Это верно для любого метода сервиса.

**Формат берётся из расширения целевого файла.** Сохранение в `avatar.png` даст PNG, в `photo.jpg` — JPEG, независимо от того, что загрузил посетитель.

**Качество** — последний аргумент, число от 0 до 100, по умолчанию 100. Оно влияет только на форматы со сжатием с потерями (JPEG, WebP); для PNG его наличие ничего не меняет.

**Целевую папку сервис не создаёт.** Если её нет, вызов закончится `ImageProcessingException` — создайте папку заранее, как в примере выше.

### Доступные операции

#### Уместить в границы

| Метод | Что делает |
|---|---|
| `saveScaledDown($source, $target, ?$width, ?$height, $quality)` | Вписывает картинку в заданные границы с сохранением пропорций. Картинку меньше границ **не увеличивает** — оставляет как есть |
| `saveConverted($source, $target, $quality)` | Та же картинка в исходном размере, перекодированная в другой формат |

Стороны в `saveScaledDown()` необязательные: `null` означает «не ограничивать». Чтобы уменьшить только по ширине, передайте только её:

```php
$this->imageProcessor->saveScaledDown($source, $target, 240);
```

{% hint style="danger" %}
Если файл нужно сохранить под фиксированным расширением, не копируйте его функцией `copy()` — для этого и нужен `saveConverted()`. Иначе в файле `.png` окажутся байты JPEG:

```php
// Неправильно: имя .png, содержимое JPEG
copy($source, $dir . 'orig.png');

// Правильно
$this->imageProcessor->saveConverted($source, $dir . 'orig.png');
```
{% endhint %}

#### Получить точный размер

Когда картинка должна занять кадр фиксированного размера — плитку в сетке, обложку, карточку каталога, — исходник почти никогда не тех же пропорций. Методы отличаются тем, чем ради этого жертвуют.

| Метод | Что делает | Чем жертвует |
|---|---|---|
| `saveCropped($source, $target, $width, $height, $position, $quality)` | Заполняет кадр целиком, лишнее обрезает по краям | Краями картинки |
| `savePadded($source, $target, $width, $height, $background, $quality)` | Помещает картинку целиком, недостающее заливает фоном | Полями по двум сторонам |
| `saveBlurredThumbnail($source, $target, $width, $height, $quality)` | Заполняет кадр размытой копией картинки, а сверху по центру кладёт её же уменьшенной | Ничем, но подложка размытая |
| `saveStretched($source, $target, $width, $height, $quality)` | Растягивает картинку до кадра | **Пропорциями** — картинка деформируется |

```php
// Квадратная обложка: обрезаем лишнее
$this->imageProcessor->saveCropped($source, $target, 150, 150);

// Карточка каталога: товар обрезать нельзя, поля белые
$this->imageProcessor->savePadded($source, $target, 400, 400, 'ffffff');
```

Последний вариант — `saveBlurredThumbnail()` — нужен там, где миниатюры стоят сеткой одинаковых плиток, а обрезать картинки жалко: размытая подложка заполняет кадр вместо пустых полей. Так сделаны миниатюры альбома и превью скриншотов в загрузках.

{% hint style="danger" %}
`saveStretched()` — единственный метод, который **не сохраняет пропорции**: картинка другой формы выйдет сплющенной или растянутой. Он нужен только тогда, когда исходник уже нужных пропорций и его требуется привести к точному размеру. Во всех остальных случаях вам нужен `saveCropped()` или `savePadded()`.
{% endhint %}

**Какую часть картинки оставит обрезка.** `saveCropped()` по умолчанию берёт центр. Если важна другая часть, передайте позицию перечислением `Johncms\Image\ImagePosition`:

```php
// У портрета внизу обычно ничего интересного, а голову обрезать нельзя
$this->imageProcessor->saveCropped($source, $target, 300, 300, ImagePosition::Top);
```

Значения: `TopLeft`, `Top`, `TopRight`, `Left`, `Center`, `Right`, `BottomLeft`, `Bottom`, `BottomRight`.

**Чем заливать поля.** Фон в `savePadded()` — любой цвет, понятный драйверу: `'fff'`, `'#ffcc00'`, `'rgb(255, 0, 0)'`. Чтобы поля остались прозрачными, передайте константу `ImageProcessorInterface::TRANSPARENT` и сохраняйте в формат с альфа-каналом — PNG или WebP:

```php
$this->imageProcessor->savePadded($source, $dir . 'cover.png', 400, 400, ImageProcessorInterface::TRANSPARENT);
```

{% hint style="warning" %}
В JPEG прозрачных пикселей не бывает: сохранив туда, вы получите поля цветом фона драйвера, а не прозрачные.
{% endhint %}

#### Водяной знак

```php
$this->imageProcessor->saveWatermarked($source, $target, UPLOAD_PATH . 'watermark.png');
```

По умолчанию знак кладётся в правый нижний угол, непрозрачным и вплотную к краю. Позиция задаётся тем же перечислением `ImagePosition`, прозрачность — числом от 0 до 100, отступ от края — в пикселях:

```php
$this->imageProcessor->saveWatermarked(
    $source,
    $target,
    $watermark,
    ImagePosition::BottomRight,
    opacity: 60,
    offset: 20
);
```

{% hint style="info" %}
Знак накладывается своего размера, под картинку он не подгоняется. Маленький логотип, растянутый на большое фото, вышел бы размытым, поэтому подготовьте файл нужного размера — или держите несколько вариантов знака под разные размеры картинок.
{% endhint %}

## Обработка ошибок

Всё, с чем сервис не справился — битый файл, неподдерживаемый формат, недоступная для записи папка, — приходит одним исключением `Johncms\Image\ImageProcessingException`.

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

Этим занимается `Johncms\Image\ThumbnailGenerator` — внедрите его через конструктор так же, как сам сервис изображений. Он возвращает путь к готовому файлу, а пересчитывает его, только если файла нет или оригинал новее:

```php
// Плитка ровно 220×300 с размытой подложкой, всегда JPEG
$preview = $this->thumbnails->blurredBackdrop($path, 220, 300);

// Уменьшенная копия, формат исходника сохраняется
$preview = $this->thumbnails->scaledDown($path, 100, 100);
```

Кэш лежит в `data/cache/thumbnails` — **вне папки сайта**, и это сделано намеренно: превью отдаёт контроллер, а значит право посетителя видеть эту картинку есть где проверить. Каталог, который веб-сервер раздаёт напрямую, такой возможности не оставляет.

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

    public function cover(Request $request, int $id): Response
    {
        $cover = $this->covers->findById($id);
        if ($cover === null) {
            return new Response('', Response::HTTP_NOT_FOUND);
        }

        try {
            $preview = $this->thumbnails->scaledDown(UPLOAD_PATH . 'covers' . DS . $cover->id . '.jpg', 220, 300);
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

`CachedImageResponse` сам проставляет `Content-Type`, `Content-Length`, `Last-Modified` и `Cache-Control: private` со сроком в неделю. Собирать `BinaryFileResponse` вручную не нужно: ядро намеренно не вызывает `Response::prepare()`, поэтому обычный файловый ответ ушёл бы с типом `text/html`.

{% hint style="info" %}
Кэш помечен `private`, а не `public`: раздающий прокси перед сайтом иначе отдал бы сохранённую копию всем подряд, включая тех, кому картинка не предназначена. Браузеру самого посетителя это не мешает — именно он и делает повторные запросы на странице, полной миниатюр.
{% endhint %}

### Никогда не берите путь из запроса

Адресуйте картинку **по идентификатору записи, которой она принадлежит**, а путь стройте в контроллере:

```php
$r->get('/my-module/preview/{id:number}', [CoverPreviewController::class, 'cover'])
    ->name('my_module.preview');
```

Ссылка вида `preview.php?img=/upload/...` — приглашение к обходу каталогов: имя с `../` внутри уводит чтение куда угодно по диску.

Если имя файла в адресе всё же неизбежно — например, нужен конкретный скриншот внутри папки, — одной меры мало. Ограничьте маршрут регулярным выражением:

```php
$r->get('/my-module/preview/{id:number}/{name}', [CoverPreviewController::class, 'screen'])
    ->name('my_module.screen_preview')
    ->requirements(['name' => '[A-Za-z0-9_.\-]+']);
```

…и **дополнительно** отрежьте путь из имени, а у результата проверьте, что он остался внутри нужной папки:

```php
public function screen(Request $request, int $id, string $name): Response
{
    $directory = UPLOAD_PATH . 'covers' . DS . 'screens' . DS . $id . DS;

    $path = realpath($directory . basename($name));
    $boundary = realpath($directory);
    if ($path === false || $boundary === false || ! str_starts_with($path, $boundary . DS)) {
        return new Response('', Response::HTTP_NOT_FOUND);
    }

    // ...дальше как в примере выше
}
```

{% hint style="info" %}
Проверок три, потому что каждая закрывает своё. Регулярное выражение маршрута отсекает очевидное, `basename()` убирает путь из имени, а сравнение с `realpath()` ловит то, что прошло мимо обеих, — например символическую ссылку, ведущую наружу.
{% endhint %}

## Драйверы и настройки

Обработка идёт через расширение **Imagick**, если оно установлено, иначе через **GD**. Выбор делается автоматически, настраивать ничего не нужно.

Правила декодирования заданы в одном месте — методе `manager()` класса `InterventionImageProcessor`:

| Настройка         | Значение  | Зачем                                                                                                            |
|-------------------|-----------|------------------------------------------------------------------------------------------------------------------|
| `autoOrientation` | включена  | Снимок с телефона хранит поворот в EXIF, а не в пикселях. Без этого фото сохранялось бы лежащим на боку          |
| `decodeAnimation` | выключена | CMS хранит один кадр. Раскладывать анимированный GIF покадрово — значит обработать каждый кадр и записать первый |
| `strip`           | включена  | Метаданные снимка содержат GPS-координаты места съёмки, а сохранённый файл публичный                             |

{% hint style="warning" %}
Автоповороту нужно расширение `ext-exif`. В поставляемом Docker-образе оно есть; если на хостинге его нет, библиотека просто не станет поворачивать картинку — ошибки не будет.
{% endhint %}

## Если нужной операции нет

Сначала посмотрите, не собирается ли нужное из того, что уже есть: несколько размеров одной картинки — это просто несколько вызовов от одного исходника.

```php
// Оригинал в разумных пределах, крупная копия и миниатюра для списка
$this->imageProcessor->saveScaledDown($source, $dir . 'orig.jpg', 1920, 1080);
$this->imageProcessor->saveScaledDown($source, $dir . 'big.jpg', 800, 600);
$this->imageProcessor->saveBlurredThumbnail($source, $dir . 'thumb.jpg', 400, 300);
```

Если этого не хватает — например, нужен поворот на заданный угол или наложение нескольких слоёв, — дальше всё зависит от того, кому эта операция пригодится.

### Операция пригодится всем

`ImageProcessorInterface` — часть ядра, и дописывать в него метод у себя не нужно: правку сотрёт следующим обновлением CMS. Предложите изменение в [репозитории JohnCMS](https://github.com/johncms/johncms) — тогда метод появится у всех и будет реализован один раз.

### Операция нужна только вашему модулю

Точки расширения, как у [политик санитайзера](html-sanitizer.md), у сервиса изображений нет. Если операция слишком специфична, чтобы просить её в ядро, модуль обрабатывает картинку сам — библиотека [Intervention Image](https://image.intervention.io/) есть в зависимостях CMS и доступна вашему коду.

{% hint style="warning" %}
У этого есть цена: модуль оказывается привязан к мажорной версии библиотеки, и её обновление в следующей версии CMS может его сломать. Поэтому держите такой код **в одном классе своего модуля** — ровно так, как ядро держит его в `InterventionImageProcessor`. Тогда починка сведётся к правке одного файла.
{% endhint %}

{% hint style="danger" %}
Не делайте обёртку, повторяющую API библиотеки: набор из `resize()`, `crop()`, `blur()` не даёт ничего — он копирует то, что оборачивает, и возвращает знание о библиотеке во все вызывающие места. Метод называется по **задаче**, а не по действию: `saveBlurredThumbnail()`, а не `crop()` плюс `blur()` плюс `insert()`.
{% endhint %}
