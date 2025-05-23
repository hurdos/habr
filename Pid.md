
<!-- # Что PID твой мне? [Название статьи] -->

История про то, как мы тюнили логи в своем проекте и как это переросло во что-то большее. Возможно кому-то это поможет в своих проектах, кому-то будет просто интересно почитать.

# Былые времена

Думаю объяснять зачем нужны логи не стоит, уверен что это должно быть очевидно каждом IT-шнику.
Но если совсем кратко, то логи нужны чтобы система сообщала, что она делает и потом, читая логи, можно было разобраться в проблемах, возникших внутри системы, ну или не разобраться, и это повод улучшить логирование.

Так вот когда-то давно система писала логи по аналогии с тем как это делается в любой *nix ОСи, т.е. записывала информацию в файл, формат записи плюс/минус был стандартный: 

`<date time> <level> <message>`

`<картинка с текстовыми логами для примера>`

Кратко о системе: это был сервис SOAP API, в логи записывались запросы, ответы и важная служебная информация.

Поиск и анализ логов был прост: открываешь файл и поиском по тексту ищешь по шаблону нужное место.

Думаю вы уже догадываетесь какие трудности возникали в результате этого.
Основная трудность - это найти в логах информацию о возникшем сбое. По сути есть только время сбоя, причем приблизительное, например "около 12ч дня наблюдали проблемы у сервиса".

Открываешь файл и начинаешь искать по времени с 12:00 по 12:10, ничего не находишь и думаешь "а был ли сбой?".

Или наоборот, конкретный случай у клиента, он в браузере увидел ошибку, система состояла из ряда сервисов, которые взаимодействовали между собой, поэтому сбой мог быть в любом из них. И приходилось смотреть по всем файлам, искать ошибки по ключевому слову `ERROR` и приблизительному времени, указанному клиентом.

С ростом нагрузки на сервис, анализ логов становился просто невозможным, начинали писать скрипты, которые парсят файлы логов в поисках ошибок и проблем. Удорожание сопровождения становилось все большим, процент времени, уделяемый командой разработки на анализ логов, стремительно возрастал. Нужно было что-то менять.

# Рождение PID

Идея с PID возникла всё из тех же предпосылок, связанных с *nix.

Техническая команда сопровождения запрашивала по возможности предоставлять запрос к системе и её ответ. Это позволяло начать поиск в логах по дате и времени, что упрощало анализ.
Однако система состояла из нескольких сервисов. Допустим, у нас есть запрос и ответ от одного сервиса — но как найти остальные логи? Теоретически это можно сделать по цепочке, но такой подход отнимал слишком много времени.

Так возникла идея связать логи единым сквозным идентификатором. Его назвали просто — PID.

Принцип работы был таким:

1. Сервис при получении запроса проверял наличие PID.
2. Если идентификатора не было — генерировал его.
3. Далее передавал этот PID всем связанным сервисам в рамках процесса.
   
Важно: PID существовал только в контексте одного внешнего процесса.

В результате в логах появился дополнительный параметр, который объединял записи всех сервисов, участвующих в обработке запроса.
Теперь, имея даже один запрос/ответ, мы могли использовать PID для быстрого поиска всех связанных логов — это заняло значительно меньше времени, чем раньше.

Позже процесс оптимизировали: вместо запроса полных данных стали использовать PID из ответа. Это дополнительно ускорило анализ логов.

# Ускорение

Тем не менее скорость анализа оставалась низкой, плюс стало очевидно, что команда разработки не может заниматься полным сопровождением/поддержкой системы. Нужен был отдел технической поддержки, где разбирали бы логи и вносили изменения в настройки системы. Многие ошибки были связанны не с багами системы, а с неверными настройками или отсутствием настроек для клиентов.

Логи в файлах становились препятствием, хоть работа с файлам кажется простой, но сопутствующие ограничения очень сильно мешали подключать ребят из тех. поддержки к анализу. Приведу ряд примеров:

* **доступы к файлам**: т.е. как минимум какой-то ssh-доступ к серверу, где есть логи;
* **размеры файлов**: требовалось отдельно научить и проинструктировать как правильно работать с по-настоящему гиганскими файлами, не правильно откроешь их и сервер накроется медным тазом;
* **медленный поиск**: из-за того что файлов могло быть много от разных сервисов, то приходилось открывать каждый и просматривать. У команды разработки были скрипты, которые они использовали для поиска, но это почти уже кодинг, а отдел ИБ сильно против того, чтобы ребята из тех. поддержки выполняли какой-то скрипт на прод-серверах.

Знаю, знаю, что все выше перечисленные ограничения можно решить, организовав отдельное хранилище с отдельными доступами и т.д. Но это тоже будет часть системы, которую надо сопровождать, на которую тоже придется тратить дополнитеные силы.

Поэтому мы решили переехать с файлов в MongoDB, т.е. записывать теперь логи в монгу.

Чем понравилась MongoDB:

* гибкий формат записи, т.е. можно расширить коллекцию дополнительными полями;
* довольно быстрая NoSQL БД;
* возможность использования индексов, PID идеально подходил под индексирование;
* возможность написания скриптов для поиска по записям.

Мы переделали механизм логирования, добавив в него возможность записывать логи в монгу. Теперь вместо файла у нас была коллекция.

Поиск по PID и по всем коллекция был в разы быстрее, вместо открытия десятков файлов, мы просто набирали запрос в монгу с указанием PID и получали все записи логов, сделанные в рамках одного процесса/одного PID'а.

Плюс ко всему получали возможность проводить анализ не по одному процессу, а по всем, указывая в скрипте шаблон поиска определенной ошибки или участка логов.

Из мелких последующих доработок следует отметить разбиение коллекций по времени, т.к. бесконечно записывать в одну коллекцию все логи сервиса было нереально. Мы добавили в название коллекции суфикс года и месяца `collection_name_202505`.

# UI тулз

Через какое-то время от тех. поддержки стали приходить жалобы, что работать с монгой не удобно, особенно, если пользуешься консолью. "Знали бы они, как не удобно было раньше" - думали мы тогда.

Первым решением было использовать Robomongo и это помогло, простой и понятный gui-клиент.

Но со временем неудобства накапливались. Например, самая частая жалоба была на xml-разметку: чтобы отформатировать ее, приходилось копировать данные в стороние форматеры, что было конечно же не удобно.

Поэтому было решено сделать свой web-интерфейс и уже его дорабатывать как хочется, и xml форматировать, и другие удобства прикручивать.

Проблема возникла откуда не ждали: отдел разработки был загружен бизнес-задачами и никак не мог начать делать UI tools.

Но как говорится "спасение утопающих - дело рук самих утопающих". Один из инженеров в тех. поддержке очень хотел стать программистом, а именно фронтенд-программистом. Решили ему дать задание сделать UI tools для чтения и анализа логов. Парень - молодец! Где-то через месяц сделал первый вариант веб-интерфейса, мы его называли "чудо-юдо". Тем не менее в работе отдела саппорта тулза прижилась.

К нашему сожалению парень, сделавший "чудо-юдо" ушел в другую контору, становится настоящим программистом. Удачи тебе герой!

А мы столкнулись с тех. долгом, но уже была основа, и отдел программистов по-немногу улучшал "чудо-юдо", превратив его со временем в мощный инструмент для тех. поддержки, но это уже другая история.

# Послесловие

Сегодня ребята из тех. поддержки, да и других отделов от разработки до админов/devops, не могут себе представить времена, когда не было UI тулзы для логов, не могут представить себе логи без PID.
