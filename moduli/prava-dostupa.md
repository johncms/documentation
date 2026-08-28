# Права доступа (permissions)

Кто что может делать на сайте, описывается **правами**. Право — это строковый ключ вида
`<модуль>.<объект>.<действие>` (`forum.post`, `users.ban.manage`), который модуль объявляет сам.
Права выдаются **ролям**, а роли — аккаунтам; редактор ролей находится в `/admin/roles`.

{% hint style="warning" %}
**Изменение в 10.0.** Раньше доступ определялся числом `users.rights` (0 — пользователь, 3 —
модератор форума, 9 — супервизор) и настройками `mod_forum`, `mod_guest`, `mod_lib`, `mod_down`,
`mod_reg`, `active`. Ни того, ни другого больше нет: число заменено ролями, настройки — правами
`forum.view`, `forum.post`, `guestbook.view`, `guestbook.post`, `library.view`, `downloads.view`,
`community.view`, `registration.register`. Сравнения с `rights` и чтение `mod_*` в стороннем
модуле нужно переписать на проверку права.
{% endhint %}

## Проверка права

В коде — через `AccessCheckerInterface`, всегда инъекцией:

```php
use Johncms\Auth\Authorization\AccessCheckerInterface;
use Johncms\Modules\Forum\Application\Services\ForumPermissions;

final readonly class PostMessageUseCase
{
    public function __construct(private AccessCheckerInterface $accessChecker)
    {
    }

    public function execute(): void
    {
        if (! $this->accessChecker->allows(ForumPermissions::POST)) {
            throw new AccessDeniedException();
        }
    }
}
```

Второй аргумент `allows()` — объект, о котором идёт речь. Он нужен, когда право зависит не только
от роли: куратор темы модерирует **свою** тему, и об этом спрашивают так:

```php
$this->accessChecker->allows(ForumPermissions::TOPIC_MODERATE, $topic);
```

В шаблонах — функция `can()`:

```twig
{% if can('forum.post') %}
    <a href="/forum/new-topic/{{ id }}/">{{ __('New topic') }}</a>
{% endif %}
```

Целый маршрут закрывается правом в объявлении маршрута, без проверки в контроллере — см.
[Маршрутизация](marshrutizaciya-routing.md).

## Объявление прав модуля

Модуль объявляет свои права провайдером — классом, реализующим `PermissionProviderInterface`.
Тег для контейнера ставится автоматически, отдельной регистрации не требуется (если каталог
`src/Application` загружается целиком).

```php
<?php

declare(strict_types=1);

namespace Mysite\Partners\Application\Services;

use Johncms\Auth\Authorization\PermissionDefinition;
use Johncms\Auth\Authorization\PermissionProviderInterface;
use Johncms\Auth\Authorization\SystemRole;

final class PartnersPermissions implements PermissionProviderInterface
{
    public const GROUP = 'partners';

    public const VIEW = 'partners.view';
    public const MANAGE = 'partners.manage';

    public function permissions(): iterable
    {
        $group = d__('partners', 'Партнёры');

        return [
            // Пятый аргумент — роли, которым право выдаётся на сайте, где его не настраивали
            // вручную. Пустой список означает «никому по умолчанию».
            new PermissionDefinition(self::VIEW, self::GROUP, d__('partners', 'Смотреть список'), $group, [
                SystemRole::Guest->value,
                SystemRole::User->value,
            ]),
            new PermissionDefinition(self::MANAGE, self::GROUP, d__('partners', 'Управлять списком'), $group),
        ];
    }
}
```

Права появляются в редакторе ролей сгруппированными по `group`, с подписью `label`. Право,
которого нет в каталоге, выдать нельзя — реестр не принимает неизвестные ключи.

После обновления, объявляющего новые права, встроенным ролям их нужно раздать:

```bash
php system/bin/console auth:sync-roles
```

Команда создаёт роли, которых у сайта ещё нет, и выдаёт встроенным ролям недостающие права по
умолчанию. Она только добавляет — то, что настроено на сайте вручную, остаётся как есть, — поэтому
запускать её после каждого обновления безопасно. То же доступно кнопкой в админке, в разделе
«Обслуживание».

## Иерархия ролей

Право отвечает на вопрос «можно ли делать это», но не на вопрос «кто кого выше». Для второго есть
уровень роли и `RoleLevels`:

```php
$targetLevel = $this->roleLevels->highestGrantedTo($userId);

if ($targetLevel > $this->roleLevels->highest($this->currentUser->identity())) {
    throw new AccessDeniedException();
}
```

Так проверяется, что модератор не редактирует профиль администратора и не банит того, кто стоит
выше. Для списка сразу многих аккаунтов есть `highestGrantedToMany()` — один запрос на страницу
вместо запроса на строку.

Роль уровня 90 и выше (`supervisor`) может всё независимо от выданных прав: это гарантированный
способ вернуться в неправильно настроенный сайт.

## Голосователи (voters)

Ответ «можно» складывается из голосов. Любой `Deny` запрещает, один `Allow` разрешает, вопрос без
голосов запрещён. Так бан перебивает роль, а токен API может урезать администратора. Модуль
добавляет своё правило классом, реализующим `AccessVoterInterface` — например, чтобы разрешить
действие автору объекта:

```php
final readonly class TopicCuratorVoter implements AccessVoterInterface
{
    public function supports(string $permission, mixed $subject): bool
    {
        return $permission === ForumPermissions::TOPIC_MODERATE && $subject instanceof ForumTopic;
    }

    public function vote(Identity $identity, string $permission, mixed $subject): Vote
    {
        return array_key_exists($identity->userId, (array) $subject->curators) ? Vote::Allow : Vote::Abstain;
    }
}
```

## Перенос настроек существующего сайта

Значения старых настроек доступа переносятся в права ролей одной командой — той же, что переводит
должности в роли:

```bash
php system/bin/console auth:migrate-legacy-access
```

Она читает `mod_forum`, `mod_guest`, `mod_lib`, `mod_down`, `mod_reg` и `active` и выдаёт (или
снимает) соответствующие права ролям `guest` и `user`. Это единственное место в системе, которое
права **снимает**: «форум только авторизованным» — это роль `guest` без `forum.view`. Именно
поэтому команда одноразовая: повторный запуск вернул бы роли к тому, что говорят старые настройки,
отменив то, что настроено в `/admin/roles` после переноса. Пока колонка `users.rights` на месте,
перенос считается незавершённым; после её удаления команда больше ничего не делает.
