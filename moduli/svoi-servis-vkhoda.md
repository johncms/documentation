# Свой сервис входа

Модуль может добавить вход через ещё один сервис — Discord, Mail.ru, Telegram, что угодно — не
трогая ядро. Маршруты, `state`, PKCE, связывание аккаунтов и создание сессии остаются в ядре;
модуль отвечает только за свой сервис.

## Типовой случай: OAuth 2.0

Наследник `AbstractOAuth2Provider` — это три адреса и маппинг полей:

```php
namespace Johncms\Modules\Discord\Auth;

use Johncms\Auth\External\AbstractOAuth2Provider;
use Johncms\Auth\External\ExternalIdentityDTO;

final class DiscordProvider extends AbstractOAuth2Provider
{
    public function key(): string
    {
        // Ключ уходит в URL, в конфиг и в таблицу user_identities. Не переименовывается.
        return 'discord';
    }

    public function label(): string
    {
        return 'Discord';
    }

    protected function authorizeUrl(): string
    {
        return 'https://discord.com/oauth2/authorize';
    }

    protected function tokenUrl(): string
    {
        return 'https://discord.com/api/oauth2/token';
    }

    protected function userInfoUrl(): string
    {
        return 'https://discord.com/api/users/@me';
    }

    protected function scope(): string
    {
        return 'identify email';
    }

    protected function mapIdentity(array $userInfo, array $token): ExternalIdentityDTO
    {
        return new ExternalIdentityDTO(
            providerUserId: (string) $userInfo['id'],
            email: $userInfo['email'] ?? null,
            emailVerified: (bool) ($userInfo['verified'] ?? false),
            nickname: $userInfo['username'] ?? null,
        );
    }
}
```

Регистрация — тегом в `config/services.php` модуля:

```php
$services->set(DiscordProvider::class)->tag('johncms.auth.external_provider');
```

Тег можно не указывать явно: ядро вешает его на всё, что реализует
`ExternalIdentityProviderInterface`.

Маршруты модулю не нужны: `/auth/discord` и `/auth/discord/callback` — ядровые и
параметризованы ключом. Ключи приложения администратор вводит в `/admin/auth/providers`, там же
берёт адрес возврата.

## Нетиповой случай

Если сервис не про authorization code flow — Telegram Login Widget (подписанный payload, никакого
`code`), Steam (OpenID 2.0), — реализуйте `ExternalIdentityProviderInterface` напрямую. Контракт
специально не привязан к OAuth2 и описывает две точки: «куда отправить пользователя» и «что
делать, когда он вернулся».

Если сервису нужно что-то ещё из callback — VK ID, например, требует `device_id` оттуда в
запросе токена — переопределите `exchangeCode()`: третьим аргументом туда приходит
`ExternalCallbackDTO`.

Провайдер **не получает `Request`**: HTTP-типы не выходят за пределы HTTP-слоя, а провайдер —
адаптер к чужому API. Контроллер ядра собирает `ExternalCallbackDTO` с параметрами запроса и
передаёт его вместе с `ExternalAuthContextDTO` (адрес возврата, `state`, PKCE-verifier).

## Про `emailVerified`

Флаг решает, будет ли аккаунт **автоматически связан** с существующим по адресу. Ставьте `true`,
только если сервис действительно подтверждает адрес. Ошибка здесь — это захват чужого аккаунта:
достаточно зарегистрироваться у провайдера с адресом жертвы.

## Переводы

Подпись кнопки — обычная строка своего gettext-домена (`d__('discord', '...')`) или просто
название сервиса, которое не переводится.

## Что нельзя испортить

Провайдер не управляет флоу: он не видит запрос, не хранит `state`, не создаёт сессию и не решает,
с каким аккаунтом связать личность. Максимум, что он может сломать, — свою же кнопку.
