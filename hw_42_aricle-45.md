### 1.
Типичное мейнстримное приложение на android является клиентом в клиент-серверной архитектуре, потому требуется частая синхронизация локальных данных с сервером.
В незрелых проектах люди напрямую работают с сетевой логикой, в более зрелых работа с сетью оформляется в виде подключаемого SDK, внутри которого реализованы и политики ретрая, кеширования, работа с сертификатами, логирование ошибок, сбор перфоманс метрик и т.д.

У нас на данный момент проект незрелый, поэтому работаем с сетью на низком уровне. 
Я бы оформил сетевой слой как подключаемый модуль, подумал над автоматизацией запросов, которые можно сделать.
Идея в том, что бекенд оформляет документ в каком-нибудь согласованном формате, при сборке проекта система сборки подгружает этот документ с описанием API и генерирует нужные методы для осуществления запроса.
Ну и помимо этого, оформил бы логику со сбором метрик, логированием и другими core частями.

### 2

Какое-то время назад я отмечал "пробел" в API нового гуглового фреймворка для построения UI. 
В их API есть функция, принимающая массив ключей, в документации уточнено, что ключи должны быть уникальны в рамках массива.
Поставлю себя на место разработчиков этого API. В текущем варианте ответственность за это требование возлагается на клиентов, однако если клиент пренебрежет правилом и не будет на своей стороне проверять, то приложение упадет и пользователю будут недовольны.
Я бы хотел избежать этой ситуации и сделать, возможно, специальную структуру данных, которая представляет собой такой же массив со всеми его особенностями, но, например, в отдельной структуре, не допускающей дубликатов, хранит множество ключей.
И это снимает ответственность с клиента, делает работу с API безопаснее, а пользователей радостней.

Так как я со своей стороны не смог повлиять на гугл и придумать решение, пришлось нам, клиентам, создавать такую структуру данных. 

### 3

В крупных android проектах часто необходимо управлять множеством конфигурационных параметров. Применение информационной избыточности позволяет хранить конфигурации в нескольких местах - например, локально на устройстве, в удаленном хранилище (как Firebase Remote Config), и в виде резервной копии в облаке. 
Это позволяет упростить API, отвечающее за управление конфигурацией, и обеспечить более надежное поведение приложения даже в случае сбоев, например, при потере соединения с удаленным сервером или санкционных блокировках.
Многие крупные компании, включая нас, сделали свой внутренний аналог сервисов Firebase (например, пуши, аутентификация, remote config), чтобы обезопасить себя.
Можно сказать, многие сделали API для работы с этими сервисами надежнее.

**Выводы**

У нас в проекте сейчас сразу несколько технических стримов идут, в ходе которых разрабыватеся некое API. 
Например, кеширование данных с вебсокета (кстати, тут немало интересных технических задач из-за того, что сервер не гарантирует порядок доставки сообщений по вебсокету).
Поэтому задание весьма важно и на ревью уже предлагал идеи по улучшению API. 
Странно, что многие, создавая некий API, даже не документируют его, не заботятся о надежности и есть множество состояний, при которых работа с API переходит в невалидное состояние.
Отсутствие инвариантов, строгих типов данных, даже отсутствие спецификаций и описаний компонент встречаются повсюду.

Из этого делается вывод, что многие не заботятся о том, насколько удобно работать с их API. Если бы еще отсутствие удобство означало строгость их кода и, как следствие, высокую надежность.. но нет.

Буду распространять необычный подход с информационной избыточностью, ведь самостоятельно до такого дойти нелегко, если не проходить курсы по CS. 
Для многих в команде странно даже, если сказать "Создаем свой тип данных с множеством операций, нужных нам". А уж создавать дополнительные структуры для целей оптимизации и упрощения работы... Ну, лично я ни разу не видел ничего подобного ни на одном проекте. 