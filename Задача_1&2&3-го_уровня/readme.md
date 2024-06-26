## Задача 1&2-го уровня

### 1、Анализ существующих UDF

Во-первых, мне нужно понять существующую реализацию функции F_WORKS_LIST. Существующий код функции выглядит следующим образом:

```sql
CREATE FUNCTION [dbo].[F_WORKS_LIST] ()
RETURNS @RESULT TABLE
(
    ID_WORK INT,
    CREATE_Date DATETIME,
    MaterialNumber DECIMAL(8,2),
    IS_Complit BIT,
    FIO VARCHAR(255),
    D_DATE varchar(10),
    WorkItemsNotComplit int,
    WorkItemsComplit int,
    FULL_NAME VARCHAR(101),
    StatusId smallint,
    StatusName VARCHAR(255),
    Is_Print bit
)
AS
BEGIN
    INSERT INTO @result
    SELECT
        W.Id_Work,
        W.CREATE_Date,
        W.MaterialNumber,
        W.IS_Complit,
        W.FIO,
        CONVERT(varchar(10), W.CREATE_Date, 104) as D_DATE,
        dbo.F_WORKITEMS_COUNT_BY_ID_WORK(W.Id_Work, 0) as WorkItemsNotComplit,
        dbo.F_WORKITEMS_COUNT_BY_ID_WORK(W.Id_Work, 1) as WorkItemsComplit,
        dbo.F_EMPLOYEE_FULLNAME(W.Id_Employee) as EmployeeFullName,
        W.StatusId,
        WS.StatusName,
        CASE
            WHEN (W.Print_Date IS NOT NULL) OR (W.SendToClientDate IS NOT NULL) 
                OR (W.SendToDoctorDate IS NOT NULL) OR (W.SendToOrgDate IS NOT NULL) 
                OR (W.SendToFax IS NOT NULL)
            THEN 1 ELSE 0
        END AS Is_Print 
    FROM Works W
    LEFT JOIN WorkStatus WS ON W.StatusId = WS.StatusID
    WHERE W.IS_DEL <> 1
    ORDER BY W.id_work DESC;
    RETURN;
END;
```

### 2、Потенциальные проблемы с производительностью

1. **Двойной подсчет подзапроса**：Функция `dbo.F_WORKITEMS_COUNT_BY_ID_WORK` вызывается дважды для подсчета количества незавершенных и завершенных рабочих элементов, в результате чего для каждого запроса выполняются два подзапроса.
2. **несколько вызовов функций**：Функция `dbo.F_EMPLOYEE_FULLNAME` вызывается один раз, что может повлиять на производительность.
3. **последовательное сканирование**：`ORDER BY W.id_work DESC` может привести к полному сканированию таблицы, особенно без соответствующих индексов.

### 3、Предложения по оптимизации

#### a. Используйте CTE, чтобы уменьшить двойной учет

Используйте общее табличное выражение (CTE), чтобы избежать двойного учета количества незавершенных и завершенных рабочих элементов:

#### Оптимизированная функция F_WORKS_LIST

```sql
CREATE FUNCTION [dbo].[F_WORKS_LIST] ()
RETURNS @RESULT TABLE
(
    ID_WORK INT,
    CREATE_Date DATETIME,
    MaterialNumber DECIMAL(8,2),
    IS_Complit BIT,
    FIO VARCHAR(255),
    D_DATE varchar(10),
    WorkItemsNotComplit int,
    WorkItemsComplit int,
    FULL_NAME VARCHAR(101),
    StatusId smallint,
    StatusName VARCHAR(255),
    Is_Print bit
)
AS
BEGIN
    WITH WorkItemsCount AS (
        SELECT 
            id_work,
            SUM(CASE WHEN is_complit = 0 THEN 1 ELSE 0 END) AS NotComplitCount,
            SUM(CASE WHEN is_complit = 1 THEN 1 ELSE 0 END) AS ComplitCount
        FROM WorkItem
        GROUP BY id_work
    )
    INSERT INTO @result
    SELECT
        W.Id_Work,
        W.CREATE_Date,
        W.MaterialNumber,
        W.IS_Complit,
        W.FIO,
        CONVERT(varchar(10), W.CREATE_Date, 104) as D_DATE,
        COALESCE(WIC.NotComplitCount, 0) as WorkItemsNotComplit,
        COALESCE(WIC.ComplitCount, 0) as WorkItemsComplit,
        E.FULL_NAME AS EmployeeFullName,
        W.StatusId,
        WS.StatusName,
        CASE
            WHEN (W.Print_Date IS NOT NULL) OR (W.SendToClientDate IS NOT NULL) 
                OR (W.SendToDoctorDate IS NOT NULL) OR (W.SendToOrgDate IS NOT NULL) 
                OR (W.SendToFax IS NOT NULL)
            THEN 1 ELSE 0
        END AS Is_Print 
    FROM Works W
    LEFT JOIN WorkStatus WS ON W.StatusId = WS.StatusID
    LEFT JOIN WorkItemsCount WIC ON W.Id_Work = WIC.id_work
    LEFT JOIN Employee E ON W.Id_Employee = E.Id_Employee
    WHERE W.IS_DEL <> 1
    ORDER BY W.id_work DESC;
    RETURN;
END;
```

### 4、 Обеспечить оптимизацию индекса

Обязательно создайте индексы для соответствующих столбцов, чтобы повысить производительность запросов:

```sql
CREATE INDEX IX_Works_Id_Work ON Works(Id_Work);
CREATE INDEX IX_WorkItem_id_work ON WorkItem(id_work);
CREATE INDEX IX_WorkStatus_StatusID ON WorkStatus(StatusID);
CREATE INDEX IX_Employee_Id_Employee ON Employee(Id_Employee);
```


### 5、Протестировать и проверить

Выполните запрос и проверьте производительность:

```sql
-- Проверьте производительность оптимизированной функции F_WORKS_LIST.
SET STATISTICS TIME ON;
SELECT TOP 30000 * FROM dbo.F_WORKS_LIST();
SET STATISTICS TIME OFF;
```

Наконец, после выполнения и тестирования оптимизированного запроса я записал время выполнения, чтобы убедиться в повышении производительности запроса.



## Задача 3-го уровня

### Возможные меры по оптимизации и их влияние

#### 1. Создайте новую таблицу
**Меры по оптимизации**: Сохраняйте некоторые данные, требующие больших вычислительных затрат, в новой таблице, например, предварительно рассчитанную статистику.

**Возможные недостатки**：
- **Синхронизация данных**: Необходимо обеспечить, чтобы при изменении исходных данных данные новой таблицы также обновлялись синхронно.Для этого могут потребоваться дополнительные триггеры или периодические пакетные задания.
- **Затраты на хранение**: Новые таблицы увеличат требования к хранилищу базы данных.
- **Повышенная сложность**: Структура базы данных становится более сложной, что может усложнить обслуживание.

**Негативное воздействие**：
- **Дополнительные затраты на производительность**: При синхронизации данных могут возникнуть дополнительные затраты на производительность, особенно при частом изменении данных.
- **Проблема согласованности**: Если возникает проблема с механизмом синхронизации, данные могут быть несогласованными.

#### 2. Добавить новый столбец
**Меры по оптимизации**: Добавление новых столбцов в существующую таблицу для хранения результатов вычислений или индексации ключевых полей.

**Возможные недостатки**：
- **Изменение структуры таблицы**: Добавление новых столбцов изменит структуру таблицы, и, возможно, потребуется обновить соответствующий код приложения.
- **Избыточность данных**: Если в новом столбце хранятся избыточные данные, это может привести к проблемам с избыточностью данных.

**Негативное воздействие**：
- **Влияние на производительность**: Изменения в структуре таблицы могут привести к сбою существующих индексов или изменению производительности запросов.
- **Затраты на обслуживание**: Необходимо обеспечить корректность данных в новой колонке и увеличить затраты на обслуживание.

#### 3. Создать триггер
**Меры по оптимизации**: Используйте триггеры для автоматического обновления результатов расчетов или синхронизации данных.

**Возможные недостатки**：
- **Накладные расходы на производительность**: Триггеры выполняются при изменении данных, что может увеличить накладные расходы на операции записи.
- **Сложность отладки**: Отладка и обслуживание триггеров относительно сложны, и найти ошибки может быть непросто.

**Негативное воздействие**：
- **Снижение производительности**: Выполнение большого количества триггеров может привести к снижению производительности операций записи, особенно в средах с высоким уровнем параллелизма.
- **Повышенная сложность**: Увеличение сложности базы данных может вызвать другие проблемы.

#### 4. Создайте хранимую процедуру или функцию
**Меры по оптимизации**: Используйте хранимые процедуры или функции для инкапсуляции сложной бизнес-логики или вычислений.

**Возможные недостатки**：
- **Затраты на техническое обслуживание**: Хранимые процедуры и функции требуют специального персонала по разработке и техническому обслуживанию.
- **Узкое место производительности**: Если хранимая процедура или функция написана неправильно, это может стать новым узким местом производительности.

**Негативное воздействие**：
- **Накладные расходы на выполнение**: Выполнение хранимой процедуры или функции может увеличить нагрузку на сервер, особенно в случае большого количества вызовов.
- **Повышенная зависимость**: Увеличивается зависимость приложения от базы данных, что может повлиять на миграцию или рефакторинг базы данных.

### Анализ примеров

Предположим, я решил создать новую таблицу для хранения предварительно рассчитанной статистики по рабочим элементам：

**Создайте новую таблицу**：
```sql
CREATE TABLE WorkItemsStats (
    Id_Work INT PRIMARY KEY,
    NotComplitCount INT,
    ComplitCount INT
);
```

**Недостатки и негативные последствия**：
- **Синхронизация данных**: Таблица "WorkItemsStats" должна обновляться при вставке, обновлении или удалении "WorkItem".
- **Затраты на хранение**: Новые таблицы увеличат требования к хранилищу базы данных.
- **Повышенная сложность**: Поддержание механизмов синхронизации и обеспечение согласованности данных требует дополнительной работы.

**Пример механизма синхронизации**：
```sql
-- Создайте триггер для обновления таблицы WorkItemsStats при вставке или обновлении WorkItem
CREATE TRIGGER trg_UpdateWorkItemsStats
ON WorkItem
AFTER INSERT, UPDATE, DELETE
AS
BEGIN
    -- Логика обновления
END;
```

Таким образом, я могу повысить производительность запросов, но это также приводит к дополнительной сложности и затратам на обслуживание.

### подводить итог

Создание новых объектов базы данных (таких как таблицы, столбцы, триггеры или хранимые процедуры) может значительно повысить производительность запросов, но у них также есть некоторые недостатки и негативные последствия.Необходимо тщательно взвесить все "за" и "против", а также внедрить соответствующие механизмы технического обслуживания и мониторинга для обеспечения общей производительности и стабильности системы.