1) Таблица tbl1, используемый движок VersionedCollapsingMergeTree, т.к. происходит удаление пары строк имеющих одинаковый первичный ключ и версию, но разный sign. Поля указанные определении таблицы должны быть указаны
и в движке в том же регистре (в примере пол Sign и Version указаын с большой буквы, соответственно в определении движка тоже должны быть указаны с большой буквы). 
CREATE TABLE tbl1
(
    UserID UInt64,
    PageViews UInt8,
    Duration UInt8,
    Sign Int8,
    Version UInt8
)
ENGINE = VersionedCollapsingMergeTree(Sign, Version)
ORDER BY UserID;

INSERT INTO tbl1 VALUES (4324182021466249494, 5, 146, -1, 1);
INSERT INTO tbl1 VALUES (4324182021466249494, 5, 146, 1, 1),(4324182021466249494, 6, 185, 1, 2);

SELECT * FROM tbl1;

SELECT * FROM tbl1 final;

2) Таблица tbl2, используемый движок ReplacingMergeTree(), т.к. есть необходимость удалять дублирующие записи по ключу сортировки.
CREATE TABLE tbl2
(
    key UInt32,
    value UInt32
)
ENGINE = ReplacingMergeTree()
ORDER BY key;

INSERT INTO tbl2 Values(1,1),(1,2),(2,1);

select * from tbl2;

3) Таблица tbl3, используемый движок ReplacingMergeTree(), с указанием параметра версии id. Движок выбран, т.к. есть необходимость удалять дублирующие записи по ключу сортировки.
CREATE TABLE tbl3
(
    `id` Int32,
    `status` String,
    `price` String,
    `comment` String
)
ENGINE = ReplacingMergeTree(id)
PRIMARY KEY (id)
ORDER BY (id, status);

INSERT INTO tbl3 VALUES (23, 'success', '1000', 'Confirmed');
INSERT INTO tbl3 VALUES (23, 'success', '2000', 'Cancelled'); 

SELECT * from tbl3 WHERE id=23;

SELECT * from tbl3 FINAL WHERE id=23;

4) Таблица tbl4, используемый движок MergeTree(), т.к. не используется аггрегация или версионирование строк.
CREATE TABLE tbl4
(   CounterID UInt8,
    StartDate Date,
    UserID UInt64
) ENGINE = MergeTree()
PARTITION BY toYYYYMM(StartDate) 
ORDER BY (CounterID, StartDate);

INSERT INTO tbl4 VALUES(0, '2019-11-11', 1);
INSERT INTO tbl4 VALUES(1, '2019-11-12', 1);

5) Таблица tbl5, используемый движок AggregatingMergeTree, т.к. используется аггрешация по ключу сортировки.
CREATE TABLE tbl5
(   CounterID UInt8,
    StartDate Date,
    UserID AggregateFunction(uniq, UInt64)
) ENGINE = AggregatingMergeTree
PARTITION BY toYYYYMM(StartDate) 
ORDER BY (CounterID, StartDate);

INSERT INTO tbl5
select CounterID, StartDate, uniqState(UserID)
from tbl4
group by CounterID, StartDate;

INSERT INTO tbl5 VALUES (1,'2019-11-12',1);

SELECT uniqMerge(UserID) AS state 
FROM tbl5 
GROUP BY CounterID, StartDate;

6) Таблица tbl6, используемый движок CollapsingMergeTree, т.к. есть необходимость удалять предыдущие строки указывая sign как -1.
CREATE TABLE tbl6
(
    `id` Int32,
    `status` String,
    `price` String,
    `comment` String,
    `sign` Int8
)
ENGINE = CollapsingMergeTree(sign)
PRIMARY KEY (id)
ORDER BY (id, status);

INSERT INTO tbl6 VALUES (23, 'success', '1000', 'Confirmed', 1);
INSERT INTO tbl6 VALUES (23, 'success', '1000', 'Confirmed', -1), (23, 'success', '2000', 'Cancelled', 1);

SELECT * FROM tbl6;

SELECT * FROM tbl6 FINAL;