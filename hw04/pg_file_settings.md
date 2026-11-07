Представление pg_file_settings — это, если можно так сказать, «рентген» ваших конфигурационных файлов. Если pg_settings показывает, что работает прямо сейчас в сервере, то pg_file_settings показывает, что написано в файлах (postgresql.conf, postgresql.auto.conf и подключённых файлах) и удастся ли это применить при следующем reload или restart.

Это ваш главный инструмент для безопасной проверки изменений до того, как вы нажмёте заветную кнопку «применить».

## Как вызвать и что означают колонки

Самый полезный запрос для начала выглядит так:
```sql
SELECT sourcefile, sourceline, seqno, name, setting, applied, error
FROM pg_file_settings
ORDER BY sourcefile, seqno;
```

Давайте разберём каждую колонку, потому что в них вся суть:

- sourcefile: Полный путь к файлу, откуда взята настройка. Это может быть postgresql.conf, postgresql.auto.conf или любой файл, подключённый через include.

- sourceline: Номер строки в этом файле. Если в error будет ошибка, вы сразу узнаете, где её искать.

- seqno: Порядковый номер обработки строки. PostgreSQL читает файлы в определённом порядке, и это поле показывает, какая настройка была прочитана раньше, а какая позже.

- name / setting: Имя параметра и значение, которое ему пытаются присвоить в файле.

- applied: Ключевая колонка. true означает, что это значение из файла будет применено (или уже применено) при reload/restart. false — что оно не будет применено.

- error: Если applied = false, здесь будет объяснение, почему значение не применилось. Если error пуст (NULL), а applied = false, это не ошибка, а признак того, что эту настройку переопределила другая, более поздняя строка для того же параметра.

## Три главных сценария использования

1. Проверка на синтаксические ошибки (самое важное)
Это то, чего вы опасались, когда говорили про ручную правку postgresql.conf. pg_file_settings позволяет увидеть опечатку до reload.

Допустим, вы случайно написали shared_buFFers или указали неверную единицу измерения 128MG. Выполняем запрос:
```sql
SELECT sourcefile, sourceline, name, setting, error
FROM pg_file_settings
WHERE error IS NOT NULL;
```

Результат может быть таким:

| sourcefile | sourceline | name  | setting | error
| --- | --- | --- | --- | --- |
| .../postgresql.conf  | 150 | work_mem | 128MG | invalid value for parameter "work_mem": "128MG"
|.../postgresql.conf       | 200        | shared_buFFers | 1GB | unrecognized configuration parameter "shared_buFFers"

Вы сразу видите файл, строку и текст ошибки. Если в файле есть хотя бы одна такая ошибка (например, нераспознанное имя параметра), сервер не будет применять настройки из этого файла целиком, и все строки из него получат applied = false.

2. Диагностика «почему не работает то, что я написал»
Вы добавили настройку в postgresql.conf, но она «не взлетела». pg_file_settings покажет все места, где встречается этот параметр:

```sql
SELECT sourcefile, sourceline, seqno, name, setting, applied
FROM pg_file_settings
WHERE name = 'log_min_duration_statement'
ORDER BY seqno;
```

Возможный результат:

| sourcefile | sourceline | seqno | name  | setting | applied
| --- | --- | --- | --- | --- | --- |
|.../postgresql.conf    | 120        | 50    | log_min_duration_statement  | 0       | f
|.../postgresql.auto.conf | 3        | 150   | log_min_duration_statement  | 500ms   | t

Здесь видно, что ваше значение 0 из postgresql.conf было переопределено более поздней записью из postgresql.auto.conf (напомню, он читается последним). У первой строки applied = f, но error пуст — это не ошибка, а именно переопределение.

3. Поиск «зависших» изменений (сравнение с pg_settings)
Это лучший способ понять, нужен ли вам reload или restart. Сравним то, что в файле, с тем, что работает сейчас:

```sql
SELECT
    s.name,
    s.setting AS current_value,
    f.setting AS file_value,
    s.context,
    s.pending_restart
FROM pg_settings s
JOIN pg_file_settings f ON s.name = f.name
WHERE s.setting != f.setting
  AND f.applied = true
  AND s.context IN ('postmaster', 'sighup');
```

Как читать результат:

- Если current_value отличается от file_value, и pending_restart = false, а context = 'sighup' → достаточно выполнить SELECT pg_reload_conf();.

- Если pending_restart = true (и context = 'postmaster') → нужен полный restart.

## Важные нюансы для начинающих

1) pg_file_settings не показывает pg_hba.conf и pg_ident.conf. Это представление только для параметров сервера (GUC). Для правил доступа есть отдельные представления: pg_hba_file_rules и pg_ident_file_mappings.

2) Это «снимок файлов», а не «состояние сервера». Если вы изменили файл, pg_file_settings сразу покажет новое значение, даже если вы ещё не сделали reload. Именно поэтому он идеален для предварительной проверки.

3) По умолчанию доступ только у суперпользователя. Если вы работаете под обычной ролью, представление может быть недоступно.

4) applied = false — не всегда ошибка. Всегда смотрите на колонку error. Пустой error при applied = false — это штатное переопределение значения более поздней строкой.

## Итог в одной фразе

pg_file_settings — это ваш «стол предпросмотра»: он показывает содержимое файлов и результат их разбора (applied, error) до того, как сервер попытается их применить, что позволяет безопасно проверять синтаксис, находить переопределения и точно понимать, нужен ли reload или restart.