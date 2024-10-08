Возьмем андроид клиент-серверное приложение по управлению устройствами умного дома.

### Ключевые свойства системы

1. Время отклика

- Описание: Время, требуемое для выполнения команды, отправленной из приложения, умным устройством в доме.
- Узкий допустимый диапазон: от 0 до 1 секунды.
- Широкий допустимый диапазон: от 0 до 5 секунд. 

2. Crash-free 

- Описание: Процент устройств, в которых не было падений приложения за месяц
- Узкий допустимый диапазон: 99,8% устройств 
- Широкий допустимый диапазон: 99% времени

3. Время синхронизации устройств

- Описание: Время, необходимое для синхронизации состояний всех подключенных устройств после изменения (например, добавление нового устройства или обновление сценария).
- Узкий допустимый диапазон: от 0 до 2 секунд.
- Широкий допустимый диапазон: от 0 до 10 секунд.

4. Успешность выполнения команд

- Описание: Процент команд пользователя, которые успешно выполняются без ошибок или сбоев.
- Узкий допустимый диапазон: 99,99% успешного выполнения команд.
- Широкий допустимый диапазон: 99% успешного выполнения команд.

5. Потребление батареи мобильным приложением

- Описание: Процент заряда батареи, расходуемый приложением за пользовательскую сессию с учетом подключения новых устройств
- Узкий допустимый диапазон: от 0% до 1% в час.
- Широкий допустимый диапазон: от 0% до 3% в час.

### Определение инвариантов для каждого свойства

1. Время отклика

- Инвариант: STABLE (Стабильный)
    - Система стремится обеспечить и поддерживать время отклика в пределах допустимого диапазона, несмотря на незначительные колебания.

2. Crash-free

- Инвариант: RESILIENT (Устойчивый)
    - Система способна восстанавливаться после сбоев и поддерживать высокий уровень crash-free, даже если внешние факторы (такие, как библиотеки внешних команд. Например, карты и геолокация) временно снижают ее. Механизм Feature toggles помогают в поддержании высокого crash-free

3. Время синхронизации устройств

- Инвариант: TRIPWIRE (Контрольный)
    - После достижения допустимого времени синхронизации система удерживает этот показатель и не допускает его ухудшения.

4. Успешность выполнения команд

- Инвариант: STRONG (Сильный)
    - Система гарантирует высокую успешность выполнения команд во всех рабочих состояниях.

5. Потребление батареи мобильным приложением

- Инвариант: STABLE (Стабильный)
    - Потребление батареи стабилизируется в пределах допустимого диапазона после первоначальной настройки или интенсивного использования приложения.


### Диапазоны инвариантов и предусловия

1. Время отклика

- Инвариант: STABLE (0–1 секунда)
- Диапазон: От 0 до 1 секунды.
- Предусловие: После установки и настройки системы время отклика стабилизируется.

2. Crash-free

- Инвариант: RESILIENT (99,8% свободных от крешей устройств)
- Диапазон: Минимум 99,8% доступности системы.
- Предусловие: Возможны короткие периоды просадок из-за внешних факторов. Помимо указанных выше, также это может быть небольшой процент раскатки новой версии на пользователей.

3. Время синхронизации устройств

- Инвариант: TRIPWIRE (0–2 секунды)
- Диапазон: От 0 до 2 секунд.
- Предусловие: После первоначальной синхронизации показатели должны поддерживаться.

4. Успешность выполнения команд

- Инвариант: STRONG (99,99% успешных команд)
- Диапазон: Не менее 99,99% команд выполняются успешно.
- Предусловие: Система функционирует в нормальных условиях.

5. Потребление батареи мобильным приложением

- Инвариант: STABLE (0%–1% в час)
- Диапазон: От 0% до 1% заряда в час.
- Предусловие: После периода адаптации приложения, когда основных модули, классы и библиотеки загрузились.


### Усиление инвариантов и добавление характеристики скорости восстановления

Усиление инвариантов:

- Crash-free
  - Возможное усиление: Переход от RESILIENT к STRONG инварианту.

- Использовать облачные технологии с гео-распределением.
    - Улучшить мониторинг системы и автоматическое переключение при сбоях.

**Потребление батареи мобильным приложением**

- Возможное усиление: Переход к более строгому STABLE инварианту с уменьшенным потреблением.
- Действия для усиления:
    - Оптимизировать код приложения для уменьшения фоновых процессов и оптимизация использования датчиков WiFi сканнеров и голосового ввода
    - Использовать энергоэффективные методы обмена данными (например, push вместо постоянного опроса).
    - Сократить использование ненужных функций при работе в фоновом режиме.

Добавление характеристики скорости восстановления:

Свойство 6: Время восстановления системы после сбоя

- Описание: Время, необходимое системе для возвращения к нормальной работе после сбоя или отказа.
- Узкий допустимый диапазон: от 0 до 30 секунд.
- Широкий допустимый диапазон: от 0 до 2 минут.
- Инвариант: RESILIENT (Устойчивый)
- Диапазон: Восстановление системы происходит в пределах установленного времени.
- Предусловие: Предусмотрены механизмы автоматического обнаружения и устранения сбоев.

Действия для улучшения скорости восстановления:

- Реализовать автоматические процедуры перезапуска сервисов.
- Использовать системы мониторинга и оповещения о сбоях.
- Внедрить отказоустойчивые архитектуры.

Усиление инварианта скорости восстановления:

- Возможное усиление: Сократить время восстановления до минимально возможного.
- Дополнительные действия:
    - Оптимизировать процессы перезапуска и синхронизации.
    - Использовать предиктивный анализ для предотвращения сбоев.
    - Обучать персонал быстрому реагированию на критические ситуации.

**Выводы**

Усиление инвариантов системы повышает ее надежность и качество работы. 
Для этого необходимо проводить оптимизацию технических и программных компонентов, внедрять современные технологии и регулярно проводить анализ производительности. 
Добавление характеристики скорости восстановления позволяет обеспечить быстрый возврат системы к нормальной работе после сбоев, что особенно важно для пользователей умного дома, полагающихся на стабильность и непрерывность работы системы.

По опыту, к сожалению, технические проекты на создание мониторингов отслеживания состояния системы реализуются не в момент создания нового приложения, а спустя продолжительное время. Иногда спустя годы. 