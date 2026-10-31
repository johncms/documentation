---
description: Как модуль отдаёт комментарии своих посетителей в общую ленту панели управления
---

# Лента комментариев

Посетители оставляют комментарии к файлам, статьям, фото, новостям — в каждом модуле свои. Чтобы не обходить модули по одному, панель собирает их в одну ленту: страница `/admin/comments` показывает все комментарии от новых к старым, блок «Последние комментарии» на главной — несколько свежих. Каждый комментарий ведёт на своё место на сайте, где на него отвечают или удаляют.

## Как это работает

Система собирает зарегистрированные источники, спрашивает у каждого его последние комментарии, сливает их по времени и режет на страницы. Общее число — сумма того, что сообщил каждый источник. У выключенного модуля источника нет, и его комментарии в ленту не попадают.

Источник отвечает только тому, кто модерирует его комментарии, для остальных он пуст. Комментарий под фото в закрытом альбоме — не дело всякого, у кого есть доступ в панель.

## Источник

Реализуйте интерфейс `Johncms\View\CommentFeed\CommentFeedSourceInterface`:

```php
<?php

declare(strict_types=1);

namespace Mysite\Blog\Application\CommentFeed;

use Johncms\Auth\Authorization\AccessCheckerInterface;
use Johncms\View\CommentFeed\CommentFeedSourceInterface;
use Johncms\View\CommentFeed\FeedComment;
use Mysite\Blog\Domain\Models\Comment;
use Mysite\Blog\Domain\Repository\CommentRepositoryInterface;

final readonly class BlogCommentFeedSource implements CommentFeedSourceInterface
{
    public function __construct(
        private AccessCheckerInterface $accessChecker,
        private CommentRepositoryInterface $comments,
    ) {
    }

    public function latestComments(int $limit): array
    {
        if (! $this->moderates()) {
            return [];
        }

        return array_values($this->comments->getLatest($limit)->map($this->toFeed(...))->all());
    }

    public function countComments(): int
    {
        return $this->moderates() ? $this->comments->countAll() : 0;
    }

    private function moderates(): bool
    {
        return $this->accessChecker->allows('blog.moderate');
    }

    private function toFeed(Comment $comment): FeedComment
    {
        return new FeedComment(
            d__('blog', 'Блог'),
            $comment->post->title,
            $comment->post->url . '#comments',
            $comment->user_id,
            $comment->user?->name ?? '',
            $comment->text,
            $comment->created_at,
        );
    }
}
```

| Метод | Что возвращает |
|---|---|
| `latestComments(int $limit)` | не больше `$limit` последних комментариев, от новых к старым |
| `countComments()` | сколько всего комментариев у источника |

Поля `FeedComment`:

| Поле | Что это |
|---|---|
| `section` | раздел сайта — метка у комментария: «Блог», «Загрузки» |
| `subject` | к чему комментарий: название записи, файла, статьи |
| `url` | где комментарии этого материала на сайте — страница, где на них отвечают и их удаляют |
| `authorId` | автор; `0` для удалённой учётной записи |
| `authorName` | имя автора — то, что было при написании, если модуль его хранит |
| `text` | текст как он лежит в базе: HTML, ничего не экранируйте |
| `time` | когда написан, unix-время |

Что важно:

* **Права проверяет источник** — и в `latestComments()`, и в `countComments()`: иначе в ленте не будет комментариев, а число страниц будет.
* **`url` — страница для человека.** Если комментарии на сайте подгружает скрипт, не давайте адрес, который отдаёт ему JSON: ведите на страницу материала, лучше сразу к блоку комментариев (`#…`).
* **Текст не экранируйте и не обрезайте.** Лента сама снимает разметку, собирает текст в одну строку и обрезает его до 300 символов; при выводе он экранируется как любой другой текст.
* **Порядок важен.** Лента сливает источники, доверяя тому, что каждый отдаёт свои комментарии от новых к старым. Добавьте к сортировке по времени сортировку по `id`, чтобы комментарии одной секунды не менялись местами.
* **Для каждой страницы ленты источник отдаёт всё до её конца** — для пятой страницы по 20 это до 100 комментариев. Запрос должен идти по индексу времени.

## Регистрация

Отдельно регистрировать источник не нужно: тег `johncms.comment_feed_source` система ставит сама — по реализованному интерфейсу, если класс объявлен с `autoconfigure()`. Для класса, объявленного отдельной строкой `set()` без `autoconfigure()`, тег указывается явно:

```php
$services->set(BlogCommentFeedSource::class)->autowire()->tag('johncms.comment_feed_source');
```

Примеры в коде — `src/Application/CommentFeed/` модулей `downloads`, `library`, `album` и `news`. Источник новостей ведёт на страницу статьи, а не на адрес, откуда её комментарии берёт скрипт.

## См. также

* [Главная панели управления](admin-dashboard.md)
* [Права доступа](prava-dostupa.md)
