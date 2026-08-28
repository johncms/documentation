---
description: Как добавить в систему ещё одну капчу, не трогая ядро
---

# Свой провайдер капчи

Модуль может добавить свою капчу — Cloudflare Turnstile, капчу от хостера, простой вопрос-ответ. Ядро трогать не нужно: провайдер появится и в формах, и на странице настроек в админке сам.

Как капча используется в формах, описано в разделе [Капча](../obshie-svedeniya/captcha.md).

## Типовой случай: чужой сервис

Все внешние сервисы устроены одинаково: виджет рисует себя по публичному ключу, кладёт токен в поле формы, а запрос на один адрес говорит, годится этот токен или нет. Это уже написано в `AbstractRemoteCaptchaProvider`, поэтому провайдер — это адрес, имена полей и разбор ответа:

```php
namespace Vasya\Turnstile\Infrastructure\Captcha;

use Johncms\Captcha\CaptchaFailure;
use Johncms\Captcha\CaptchaResult;
use Johncms\Captcha\Providers\AbstractRemoteCaptchaProvider;

final class TurnstileProvider extends AbstractRemoteCaptchaProvider
{
    public function key(): string
    {
        // Ключ уходит в конфигурацию. Не переименовывается: сайт, где он записан
        // в captcha.local.php, после переименования незаметно получит другую капчу.
        return 'turnstile';
    }

    public function label(): string
    {
        return 'Cloudflare Turnstile';
    }

    public function fieldName(): string
    {
        // Имя поля, в котором виджет присылает токен.
        return 'cf-turnstile-response';
    }

    protected function verifyUrl(): string
    {
        return 'https://challenges.cloudflare.com/turnstile/v0/siteverify';
    }

    protected function template(): string
    {
        return '@turnstile/public/widget.twig';
    }

    protected function verifyPayload(string $answer, string $secret, ?string $clientIp): array
    {
        $payload = ['secret' => $secret, 'response' => $answer];

        if ($clientIp !== null) {
            $payload['remoteip'] = $clientIp;
        }

        return $payload;
    }

    protected function judge(array $response): CaptchaResult
    {
        if (($response['success'] ?? false) === true) {
            return CaptchaResult::passed();
        }

        // Коды ошибок сервиса написаны для того, кто его настраивал, поэтому их пишут
        // в журнал, а посетителю показывают понятную фразу.
        $this->logRefusal($response);

        return CaptchaResult::failed(CaptchaFailure::Mismatch);
    }
}
```

Базовый класс уже делает HTTP-запрос с таймаутом, превращает недоступность сервиса в `CaptchaFailure::Unavailable`, отказывается проверять что-либо без ключей и объявляет два поля настроек — публичный и секретный ключ. Если сервис называет их иначе, переопределите `siteKeyLabel()` и `secretKeyLabel()`.

### Шаблон виджета

Лежит в шаблонах модуля, в его пространстве имён:

```twig
{#
    @var captcha \Johncms\Captcha\CaptchaChallenge
    @var errors  array
#}
<div class="form-group">
    <div class="cf-turnstile" data-sitekey="{{ captcha.params.site_key }}"></div>
    {% include '@theme/components/field-errors.twig' with {errors: errors} only %}
</div>
<script src="https://challenges.cloudflare.com/turnstile/v0/api.js" async defer></script>
```

### Регистрация

Ядро вешает тег `johncms.captcha_provider` на всё, что реализует `CaptchaProviderInterface`, поэтому отдельной регистрации не требуется — при условии, что сервис объявлен с `autoconfigure()`:

```php
$services->set(TurnstileProvider::class)->autowire()->autoconfigure();
```

Если autoconfigure не используется, укажите тег явно:

```php
$services->set(TurnstileProvider::class)->tag('johncms.captcha_provider');
```

После этого провайдер появляется в списке на `/admin/settings/captcha` — со своими полями настроек и предупреждением, пока в них не введены ключи.

## Свои настройки

Всё, что провайдеру нужно спросить у администратора, он описывает сам — страница настроек рисуется по этому описанию, собственного шаблона провайдеру не нужно:

```php
use Johncms\Captcha\CaptchaSettingField;
use Johncms\Captcha\CaptchaSettingType;

protected function extraSettingsFields(): array
{
    return [
        new CaptchaSettingField(
            key: 'theme',
            type: CaptchaSettingType::Select,
            label: d__('turnstile', 'Оформление виджета'),
            default: 'auto',
            options: ['auto' => 'Auto', 'light' => 'Light', 'dark' => 'Dark'],
        ),
    ];
}
```

Типы полей: `Text`, `Password`, `Number`, `Checkbox`, `Select`. Значение читается там, где оно нужно:

```php
$theme = $this->options()->string('theme', 'auto');
```

{% hint style="warning" %}
Секреты объявляйте типом `Password`. Такое поле никогда не отдаётся обратно в браузер: страница показывает лишь то, что значение сохранено. Пустое поле при сохранении означает «оставить как было», а не «стереть».
{% endhint %}

## Нетиповой случай

Если капча не про «токен и запрос на проверку» — картинка своего образца, вопрос-ответ, проверка по внутреннему списку, — реализуйте `CaptchaProviderInterface` напрямую. Контракт специально описывает всего две вещи: что показать форме и что думать про присланный ответ.

```php
public function challenge(string $scope): CaptchaChallenge;

public function verify(string $answer, string $scope, ?string $clientIp = null): CaptchaResult;
```

`$scope` — имя формы. Если капча что-то запоминает между показом и проверкой (как встроенная — код в сессии), храните это под ключом, куда входит область: иначе две формы, открытые в двух вкладках, затрут ответы друг друга.

## Что провайдер не получает

**`Request`.** HTTP-типы не выходят за пределы HTTP-слоя, а провайдер — адаптер к чужому API. Адрес посетителя, если он нужен сервису, приходит третьим аргументом `verify()`.

**Решение о том, показывать ли капчу.** Это решает форма. Провайдер отвечает на два вопроса и не управляет ни флоу, ни доступом.

## Причины отказа

Возвращайте ту, которая соответствует случаю, — от неё зависит и текст для посетителя, и то, как это будет выглядеть в журнале:

| Причина | Когда |
|---------|-------|
| `Missing` | ответа нет |
| `Mismatch` | ответ неверный |
| `Expired` | задание устарело или уже было отвечено |
| `LowScore` | сервис счёл посетителя ботом, не дав ему задания |
| `Unavailable` | сервис недоступен или отказал |
| `NotConfigured` | не хватает ключей |

Отдельная `Unavailable` — не формальность: сообщение «проверочный код введён неверно» в ответ на упавший сервис отправит посетителя перебирать варианты, которых у него нет.
