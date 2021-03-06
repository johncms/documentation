# Структура модуля

Модули располагаются в папке **modules**  
Обычно модуль для JohnCMS имеет следующую структуру:

* modules
  * module\_name
    * Controllers
    * Install
    * locale
    * templates

Данная структура носит лишь рекомендательный характер и не является обязательной.  
Система не накладывает ограничений на разработчика и разработчик вправе использовать свою структуру модуля, которая для него будет удобнее. 

Давайте подробнее посмотрим на структуру и разберемся что и для чего предназначено.

* **modules** - это обычная системная папка с модулями.
  * **module\_name** - это папка с названием модуля \(например forum, community и т.п.\)
    * **Controllers** - папка с контроллерами.
    * **Install** - папка с установочными файлами
    * **locale** - это папка в которой хранятся файлы локализации модуля. Если модуль мультиязычный, то эта папка обычно есть.
    * **templates** - в этой папке хранятся шаблоны модуля.

Директория с названием модуля используется как пространство имен для автоматической загрузки классов.

