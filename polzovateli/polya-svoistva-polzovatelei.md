# Поля \(свойства\) пользователей

Для работы с пользователями в JohnCMS используется класс **`\Johncms\Users\User()`**  
У пользователя есть различные свойства \(поля\).

## Основные свойства пользователя

Список основных свойств пользователя, которые есть в таблице users:

| Название поля | Описание |
| :--- | :--- |
| name | Логин пользователя |
| name\_lat | Логин, но в нижнем регистре, латиницей |
| password | Хэш пароля пользователя |
| rights | Права пользователя. Может содержать одно из следующих значений: **0** - Обычный пользователь **3** - Модератор форума **4** - Модератор загрузок **5** - Модератор библиотеки **6** - Супермодератор **7** - Администратор **9** - Супервизор |
| failed\_login | Количество неудачных попыток авторизации |
| imname | Имя |
| sex | Пол пользователя. Содержит одно из следующих значений: **m** - Мужчина **zh** - Женщина |
| komm | Количество комментариев |
| postforum | Количество постов на форуме |
| postguest | Количество постов в гостевой |
| yearofbirth | Год рождения |
| datereg | Дата регистрации \(timestamp\) |
| lastdate | Дата последнего визита \(timestamp\) |
| mail | E-mail адрес |
| icq | ICQ \(устаревшее\) |
| skype | Skype |
| jabber | Jabber \(устаревшее\) |
| www | Сайт пользователя |
| about | О себе |
| live | Город, страна проживания |
| mibile | Номер телефона |
| status | Статус пользователя |
| ip | IP адрес В таблице хранится в преобразованном формате \(ip2long\). Если вы получаете и записываете данные с помощью класса **\Johncms\Users\User\(\)**, ****вам не нужно заботиться о преобразовании. Вы будете видеть IP в обычном формате. Все преобразования выполняются автоматически. |
| ip\_via\_proxy | IP адрес за прокси \(если удалось определить\) В таблице хранится в преобразованном формате \(ip2long\). Если вы получаете и записываете данные с помощью класса **\Johncms\Users\User\(\)**, ****вам не нужно заботиться о преобразовании. Вы будете видеть IP в обычном формате. Все преобразования выполняются автоматически. |
| browser | User Agent. Если используете модель **\Johncms\Users\User\(\)**, то поле будет в безопасном для вывода виде. Дополнительно экранировать не требуется. |
| preg | Пометка подтвержденного пользователя. Если поле запрашивается из модели, то оно будет содержать **boolean** значение \(**true/false**\). В таблице хранится число 0 или 1 |
| regadm | Логин администратора, который подтвердил регистрацию пользователя |
| mailvis | Пометка включенного отображения e-mail адреса в профиле. Если поле запрашивается из модели, то оно будет содержать **boolean** значение \(**true/false**\). В таблице хранится число 0 или 1 |
| dayb | День рождения |
| monthb | Месяц рождения |
| sestime | Текущее время активности пользователя \(время активности сессии\) |
| total\_on\_site | Сколько провёл на сайте \(устаревшее и не используется\). |
| lastpost | Время последнего поста \(timestamp\) |
| rest\_code | Код восстановления пароля |
| rest\_time | Время восстановления пароля |
| movings | Количество переходов по страницам в рамках текущей сессии. |
| place | Местоположение пользователя |
| set\_user | Настройки пользователя. При запросе этого поля из модели содержит объект класса **Johncms\System\Users\UserConfig** При записи через модель, принимает обычный массив и автоматически преобразует в нужный формат. В таблице данные хранятся в сериализованном виде. **Поля доступные в объекте:** **directUrl** - Прямые ссылки **fieldHeight** - Высота полей ввода **kmess** - Количество элементов на страницу **lng** - Выбранный язык **timeshift** - Сдвиг времени **youtube** - Youtube плеер   |
| set\_forum | Настройки форума. Массив с настройками форума. Может быть пустым, если пользователь не сохранял настройки. |
| set\_mail | Настройки почты. Массив с настройками почты. Может быть пустым, если пользователь не сохранял настройки. |
| karma\_plus | Количество положительных голосов в карме |
| karma\_minus | Количество отрицательных голосов в карме |
| karma\_time | Время голосования в карме |
| karma\_off | Запрет кармы. Если поле запрашивается из модели, то оно будет содержать **boolean** значение \(**true/false**\). В таблице хранится число 0 или 1 |
| comm\_count | Количество комментариев |
| comm\_old | Устаревшее, не используется |
| smileys | Подборка смайлов пользователя. Массив. Может быть пустым, если пользователь не добавлял смайлы в подборку. |
| notification\_settings | Настройки уведомлений. Массив с настройками уведомлений. |

  
Модель **`\Johncms\Users\User()`** в дополнение к основным полям возвращает дополнительные вычисленные поля.

## **Дополнительные свойства**

Список дополнительных свойств пользователя:

| Название поля | Описание |
| :--- | :--- |
| is\_online | Метка пользователя онлайн \(**true/false**\) |
| rights\_name | Название прав доступа текущего пользователя \(для обычных пользователей пустая строка\) |
| profile\_url | Ссылка на страницу просмотра профиля пользователя |
| search\_ip\_url | Ссылка на страницу поиска по ip |
| whois\_ip\_url | Ссылка на страницу whois ip |
| search\_ip\_via\_proxy\_url | Ссылка на страницу поиска по IP за прокси |
| whois\_ip\_via\_proxy\_url | Ссылка на страницу whois IP за прокси |
| ban | Массив активных банов пользователя |
| is\_valid | Свойство используется при работе от текущего пользователя. **true** - если пользователь авторизован и подтвержден. **false** - если пользователь не авторизован или не подтвержден. |
| is\_birthday | **true** - если у пользователя день рождения. **false** - если нет. |
| birthday\_date | Т.к. дата рождения в таблице users хранится в отдельных полях, то при запросе этого свойства она собирается в одну строку. |
| display\_place | Местоположение пользователя для отображения. Содержит html код ссылки на страницу. |
| formatted\_about | Обработанное поле "О себе". bb-коды преобразованы в html код. |
| website | Обработанное поле "Сайт". bb-коды преобразованы в html код. |
| last\_visit | Дата последнего визита в человекопонятном виде. Обратите внимание, если пользователь сейчас онлайн, это свойство будет пустым. |
| photo | Фотография пользователя. Если фотографии нет, возвращает пустой массив. Если фотография есть, возвращает массив со ссылками на фото: **photo** - Большая фотография. **photo\_preview** - Маленькая фотография для предпросмотра. |
