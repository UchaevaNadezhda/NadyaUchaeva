= Паспорт микросервиса sme-gl-afaml-operations
:toc:
// остальные атрибуты в этом заголовке - это кастомные переменные, которые будут использоваться дальше в ТЗ
:jira_url: https://dit.rencredit.ru/jira/browse

// src_path - относительный путь, по которому расположены другие документы, которые далее будут включаться в ТЗ, схема plantuml, и прочие документы
:src_path: .

:doc_path: {src_path}/annex
:tip: 💡
// yaml_file - имя файла с контрактом МкС в каталоге src_path
:yaml_file: service-api.yaml

//common_errors_url - адрес страницы в Confluence с возможных кодов ошибок. Используется далее в описании исключительных сценариев
:common_errors_url: https://dit.rencredit.ru/confluence/pages/viewpage.action?pageId=112132821

// adlayer-134_url - адрес страницы с adlayer-134
:adlayer-134_url: https://dit.rencredit.ru/confluence/pages/viewpage.action?pageId=52396409

// adlayer-105_url - адрес страницы с adlayer-105
:adlayer-105_url: https://dit.rencredit.ru/confluence/pages/viewpage.action?pageId=52986556

// sme-check-operation_url - адрес страницы микросервиса в Confluence
:sme-gl-afml-operations_url: https://dit.rencredit.ru/confluence/display/MIC/sme-gl-afaml-operations_1.0.0

// sme-check-operation_url - адрес страницы микросервиса в Confluence
:sme-check-operation_url: https://dit.rencredit.ru/confluence/display/MIC/sme-check-operation_1.2.0

// sme-info-operation_url - адрес страницы микросервиса в Confluence
:sme-info-operation_url: https://dit.rencredit.ru/confluence/display/MIC/sme-info-operation_1.2.0

// JIRA
:jira: https://dit.rencredit.ru/jira/browse/

// Ссылки на JSON схемы
:json_schemes: {src_path}/annex/json-schemas

.История изменений
[cols="1a,1a,2a,5a,2a,2a", options="header"]

|===
|Версия

док.|Версия

МкС (должна совпадать с версией контракта!)|Дата|Описание| Задачи в Jira | Автор

| 1
| 1.0.0
| 24.03.2023
|Первичная версия микросервиса:

. Описан метод `sentPaymentForCheck` - Получение из БД и отправка на проверку входящего платежа.

| {jira}BRF-25077[BRF-25077]

'''

Epic: {jira}INT-3255[INT-3255]

'''

Задача: {jira}INT-6988[INT-6988]

| mailto:ext_alezov2@rencredit.ru[Лёзов А.П.]

| 1
| 1.0.0
| 14.04.2023
|Первичная версия микросервиса:

. Описан метод `sentPaymentForInfo` - Получение из БД и отправка в MQ через мкс. sme-info-operations info-сообщения по проверенному входящему платежу.

| {jira}BRF-25077[BRF-25077]

'''

Epic: {jira}INT-3255[INT-3255]

'''

Задача: {jira}INT-7563[INT-7563]

| mailto:ext_alezov2@rencredit.ru[Лёзов А.П.]

|===



== Общие сведения
.Общие сведения о микросервисе
[width= "80%", cols="1a,2a", options="header"]

|===

|Параметр |Описание
|Техническое название сервиса	| sme-gl-afaml-operations
|Назначение сервиса	| Микросервис для получения из БД Diasoft (GL) входящих платежей и передачи их на проверку в систему SAS (AF/AML) с последующим обновлением БД.
|Используется системами	| мкс. sme-check-operations, мкс. sme-info-operations
|Использует системы	| Diasoft (GL, интеграционные таблицы)
|Протокол передачи данных	| JDBC\HTTPS
|Тип сервиса	|  адаптер к БД
|Кодировка	| UTF-8
|Статус	|
* [x] В разработке
* [ ] На среде TEST
* [ ] На среде PREPROD
* [ ] На среде PROD
* [ ] Не используется
|===

== Контракт
link:{src_path}/{yaml_file}[{yaml_file}]


== Назначение
МкС предназначен для выполнения следующих функций:

* Получение из БД списка входящих платежей и отправка их в мкс. sme-check-operations для проверки в SAS (check). Запись в БД результата проверки в SAS.
* Получение из БД списка входящих платежей и отправка их в мкс. sme-info-operations для уведомления SAS об исполнении операции (info). 

== Детальное описание методов сервиса
=== [[SM_01]]SM_01. Получение списка входящих платежей, отправка их на проверку в SAS и обновление БД по результату проверки.
*Название метода*:  sentPaymentForCheck +
*Путь метода*: нет +
*Тип взаимодействия*: синхронный, +
*Тип операции*: SELECT\UPDATE +
*Тип входящего сообщения*: нет +
*Тип исходящего сообщения*: http ответ, Content-Type: application/json +
*Влияние на группы продуктов*: не применимо +
*Описание очередей MQ*:  +
*Имя ресурса*:

==== Краткое описание метода
Метод предназначен для получения из БД Diasoft входящих платежей через JDBC соединение, отправку их на проверку в SAS (AF и AML) через мкс. sme-check-operations, получение ответа о результате проверки и обновления через JDBC в БД статуса по результату проверки.

=== Диаграмма работы метода
image::./annex/diagrams/SM_01.puml[]
(См _annex/diagrams/SM_01.puml_ в боковой панели). 

==== Алгоритм работы метода
. Запуск метода sentPaymentForCheck согласно расписанию с периодичностью <<config-params, RunPeriod>>. 
. Открывается транзакция к БД (MS SQL). Параметры подключения указаны в <<sreda, табл. Среды подключения>>. 
.. Если подключение к БД открыть не удалось, то в лог пишется {common_errors_url}[EC-Exception]. Сценарий завершается.
. Выполняется SELECT-запрос с ограничением количества записей согласно <<config-params, NumberOfStrings>> со следующей выборкой:
[%collapsible]
====
[source,sql]
----
SELECT ID, DOCNUMBER, BATCH, DOCDATE, OPCODE, PAYMENTPURPOSE, CURRENCYDEB, CURRENCYCRE, AMOUNTDEB, AMOUNTCRE, AMOUNTRUB, TYPE, USERLOGINOPEN, DEALTRANSACTID, GLOBALID, PAYERBANKBIC, PAYERBANKCORRACCOUNT, PAYERACCOUNT, PAYERINN, PAYERNAME, PAYEECIFID, PAYEEACCOUNT, PAYEEINN, SYSTEMNAME, MSGTS, FLOWTYPE, MSGTYPE, RESPONSECODE, STATUSCODE, STATUSCHANGETIME, ORIGTS
FROM SME_AFAML_DocAttr_Inbox
WHERE RESPONSECODE IS NULL OR RESPONSECODE = ''?
ORDER BY ID ASC
OFFSET 0 ROWS FETCH NEXT 'NumberOfStrings' ROWS ONLY
----
====

.. Если SELECT-запрос выполнить не удалось или SELECT-запрос выполнен с ошибкой, то в лог пишется {common_errors_url}[EC-Exception]. Сценарий завершается.
.. Если SELECT-запрос пустой, то в лог пишется HTTP Status 204 No Content. Сценарий завершается.
.. Если SELECT-запрос не пустой, то:
... По каждой записи из SELECT-запроса в цикле выполняется следующий порядок действий:
.... Выполняется преобразование <<SM_01_T_01, SM_01_T_01>>.
.... Вызывается метод POST SMEIncomingPayment мкс. link:{src_path}/{sme-check-operation_url}[sme-check-operations].
..... Если в ответ возвращается HTTP Status 400 или 500, то выполняется преобразование <<SM_01_T_03, SM_01_T_03>>.
..... Если в ответ возвращается HTTP Status 200, то выполняется преобразование <<SM_01_T_02, SM_01_T_02>>.
.... По результату преобразований выполняется запрос UPDATE:
[%collapsible]
====
[source,sql]
----
UPDATE SME_AFAML_DocAttr_Inbox SET RESPONSECODE = {responseCode}, EXTCOMMENT = {extComment}, INTCOMMENT = {intComment}, PROCESSTS = {Server date-time}
WHERE ID = {ID}
----
====

..... Если UPDATE-запрос выполнить не удалось или UPDATE-запрос выполнен с ошибкой, то в лог пишется {common_errors_url}[EC-Exception]. Сценарий переходит на следующий шаг.
..... Если запись из запроса SELECT не последняя, то алгоритм переходит к следующей записи из запроса SELECT пока UPDATE не будет сделан по последней записи.
. Транзакция с БД закрывается. В лог возвращается ответ HTTP Status 200. Основной сценарий завершается.

=== [[SM_02]]SM_02. Получение списка входящих платежей для отправки по ним в SAS info-сообщения о выполнении операции. Обновление записи в БД при успешной передачи info-сообщения.
*Название метода*:  sentPaymentForInfo +
*Путь метода*: нет +
*Тип взаимодействия*: синхронный, +
*Тип операции*: SELECT\UPDATE +
*Тип входящего сообщения*: нет +
*Тип исходящего сообщения*: http ответ, Content-Type: application/json +
*Влияние на группы продуктов*: не применимо +
*Описание очередей MQ*:  +
*Имя ресурса*:

==== Краткое описание метода
Метод предназначен для получения из БД Diasoft входящих платежей через JDBC соединение и отправку по ним info-сообщения в SAS (AF и AML) через мкс. sme-info-operations, обновления через JDBC в БД записи при успехе.

=== Диаграмма работы метода
image::./annex/diagrams/SM_02.puml[]
(См _annex/diagrams/SM_02.puml_ в боковой панели). 

==== Алгоритм работы метода
. Запуск метода sentPaymentForInfo согласно расписанию с периодичностью <<config-params, RunPeriod>>. 
. Открывается транзакция к БД (MS SQL). Параметры подключения указаны в <<sreda, табл. Среды подключения>>. 
.. Если подключение к БД открыть не удалось, то в лог пишется {common_errors_url}[EC-Exception]. Сценарий завершается.
. Выполняется SELECT-запрос с ограничением количества записей согласно <<config-params, NumberOfStrings>> со следующей выборкой:
[%collapsible]
====
[source,sql]
----
SELECT ID, DOCNUMBER, BATCH, DOCDATE, OPCODE, PAYMENTPURPOSE, CURRENCYDEB, CURRENCYCRE, AMOUNTDEB, AMOUNTCRE, AMOUNTRUB, TYPE, USERLOGINOPEN, DEALTRANSACTID, GLOBALID, PAYERBANKBIC, PAYERBANKCORRACCOUNT, PAYERACCOUNT, PAYERINN, PAYERNAME, PAYEECIFID, PAYEEACCOUNT, PAYEEINN, SYSTEMNAME, MSGTS, FLOWTYPE, MSGTYPE, RESPONSECODE, STATUSCODE, STATUSCHANGETIME, ORIGTS, WAITFLAG, PROCESSTS
FROM SME_AFAML_DocAttr_Inbox
WHERE FLOWTYPE = 'info' AND WAITFLAG = '1' OR WAITFLAG IS NULL OR WAITFLAG = ''?
ORDER BY ID ASC
OFFSET 0 ROWS FETCH NEXT 'NumberOfStrings' ROWS ONLY
----
====

.. Если SELECT-запрос выполнить не удалось или SELECT-запрос выполнен с ошибкой, то в лог пишется {common_errors_url}[EC-Exception]. Сценарий завершается.
.. Если SELECT-запрос пустой, то в лог пишется HTTP Status 204 No Content. Сценарий завершается.
.. Если SELECT-запрос не пустой, то:
... По каждой записи из SELECT-запроса в цикле выполняется следующий порядок действий:
.... Выполняется преобразование <<SM_02_T_01, SM_02_T_01>>.
.... Вызывается метод POST infoSMEPaymentTransfer мкс. link:{src_path}/{sme-info-operation_url}[sme-info-operation].
..... Если в ответ возвращается HTTP Status 400 или 500, то выполняется UPDATE1:
[%collapsible]
====
[source,sql]
----
UPDATE SME_AFAML_DocAttr_Inbox SET WAITFLAG = '8'
WHERE ID = {ID}
----
====
..... Если в ответ возвращается HTTP Status 200, то выполняется UPDATE2: 
[%collapsible]
====
[source,sql]
----
UPDATE SME_AFAML_DocAttr_Inbox SET WAITFLAG = '9'
WHERE ID = {ID}
----
====
..... Если UPDATE-запрос выполнить не удалось или UPDATE-запрос выполнен с ошибкой, то в лог пишется сообщение "UPDATE по записи {ID} выполнить не удалось. Сценарий переходит на следующий шаг.
..... Если запись из запроса SELECT не последняя, то алгоритм переходит к следующей записи из запроса SELECT пока UPDATE не будет сделан по последней записи.
. Транзакция с БД закрывается. В лог возвращается ответ HTTP Status 200. Основной сценарий завершается.


== Конфигурационные параметры
[[config-params]]
|===
|Код	|Значение	|Комментарий

|RunPeriod	|30 сек.	|Периодичность запуска метода sentPaymentForCheck
|NumberOfStrings	|250	|Количество строк, возвращаемое запросом SELECT метода sentPaymentForCheck
|===

== Среды подключения
[[sreda]]
|===
|Наименование среды	|Параметры подключения	|Комментарий

|DEV	|Сервер БД: SRVTST8141\diasoftglDEV Алиас: DiasoftGLdev URL: jdbc:jtds:sqlserver://SRVTST8141:1433/DiasoftGLDev УЗ: svc_sme_sasmcs пароль: rPBHaaVNJ8BETaF4dE0n	|DEV-среда
|TEST	|Сервер БД: SRVTST14725\DiasoftGLtest Алиас: DiasoftGLtest URL: jdbc:jtds:sqlserver://diasoftGLtest.rccf.ru:1433/DiasoftGLtest УЗ: svc_sme_sasmcs пароль: rPBHaaVNJ8BETaF4dE0n	|TEST-среда
|PREPROD-FT	|	|PREPROD-FT-среда
|PREPROD-NT	|	|PREPROD-NT-среда
|PROD	|	|PROD-среда

|===

== Преобразование данных
[[SM_01_T_01]]
=== SM_01_T_01 - Преобразование запроса к методу SMEIncomingPayment мкс. sme-check-operations из результата SELECT-запроса.
.Таблица преобразования 1
[%collapsible]
====
include::{src_path}/annex/transformation_tables/SM_01_T_01.adoc[]
====

[[SM_01_T_02]]
=== SM_01_T_02 - Преобразование ответа от  метода SMEIncomingPayment мкс. sme-check-operations для формирования UPDATE-запроса.
.Таблица преобразования 2
[%collapsible]
====
include::{src_path}/annex/transformation_tables/SM_01_T_02.adoc[]
====

[[SM_01_T_03]]
=== SM_01_T_03 - Преобразование ответа от  метода SMEIncomingPayment мкс. sme-check-operations при ошибках и\или таймауте для формирования UPDATE-запроса.
.Таблица преобразования 3
[%collapsible]
====
include::{src_path}/annex/transformation_tables/SM_01_T_03.adoc[]
====

[[SM_02_T_01]]
=== SM_02_T_01 - Преобразование полей запроса SELECT для вызова метода infoSMEPaymentTransfer мкс. sme-info-operation.
.Таблица преобразования 4
[%collapsible]
====
include::{src_path}/annex/transformation_tables/SM_02_T_01.adoc[]
====

== Нефункциональные требования
=== Протоколирование работы
. Результаты работы сервиса должны логироваться.
.. Способ логирования: {adlayer-105_url}[стандартный механизм]

=== Производительность
|===
h|Параметр	h|Значение

|Максимальное количество запросов в день	|
|Пиковая нагрузка|
|Максимальное время отклика	|
|===

=== Доступность
. Время доступности: 24x7*365

=== Класс восстановления
|===
|Класс критичности	|Класс восстановления |RTO |RPO

| Mission Critical|1 класс |4 часа |1,5 часа
|===

