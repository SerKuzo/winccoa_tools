# Установка SQLite ODBC на Astra Linux

Инструкция проверена на Astra Linux **1.8.5.46** (x86_64) с `unixODBC 2.3.11`.

В этой схеме используется SQLite ODBC Driver **0.99991**, собранный из исходников.

## 1. Проверить систему и пакеты

```bash
uname -m
cat /etc/astra/build_version
apt-cache policy unixodbc libsqliteodbc sqliteodbc
```

Если пакет `libsqliteodbc` отсутствует в репозиториях Astra, установите инструменты сборки:

```bash
sudo apt update
sudo apt install build-essential unixodbc-dev libsqlite3-dev
```

`wget` нужен только для скачивания архива:

```bash
sudo apt install wget
```

## 2. Получить архив

Архив можно скачать заранее на другой машине и перенести в `/tmp`.

Ожидаемое имя файла:

```text
sqliteodbc-0.99991.tar.gz
```

Проверьте, что файл находится на Astra:

```bash
ls -lh /tmp/sqliteodbc-0.99991.tar.gz
```

Если HTTPS-проверка сайта не проходит из-за ошибки сертификата, не отключайте проверку сертификата на промышленной системе. Скачайте архив на доверенной машине и передайте его на Astra через SFTP/SCP.

## 3. Распаковать исходники

```bash
cd /tmp
tar -xzf sqliteodbc-0.99991.tar.gz
cd sqliteodbc-0.99991
```

Проверить содержимое архива:

```bash
ls
```

## 4. Собрать и установить драйвер

```bash
./configure --prefix=/usr/local
make -j"$(nproc)"
sudo make install
```

Предупреждение `libtool: install: warning: remember to run libtool --finish` не является ошибкой для этой установки.

После установки драйвер находится здесь:

```text
/usr/local/lib/libsqlite3odbc.so
```

Проверка:

```bash
ls -l /usr/local/lib/libsqlite3odbc.so
ldd /usr/local/lib/libsqlite3odbc.so
```

## 5. Зарегистрировать драйвер в unixODBC

Создайте или отредактируйте `/etc/odbcinst.ini` от имени root:

```ini
[SQLite3]
Description=SQLite3 ODBC Driver
Driver=/usr/local/lib/libsqlite3odbc.so
```

Проверьте регистрацию:

```bash
odbcinst -q -d
```

В выводе должна быть строка:

```text
[SQLite3]
```

## 6. Создать DSN для базы WinCC OA

Создайте или отредактируйте `/etc/odbc.ini`:

```ini
[WinccSQLite]
Driver=SQLite3
Database=/opt/scadcc_proj/mns/db/messages.db
```

Проверьте, что файл базы существует:

```bash
ls -l /opt/scadcc_proj/mns/db/messages.db
```

Пользователь, от которого запускается WinCC OA Manager/CTRL, должен иметь права чтения базы и записи в каталог базы. SQLite может создавать рядом файлы журнала и блокировки.

## 7. Проверить подключение через isql

```bash
isql -v WinccSQLite "" ""
```

В интерактивном режиме выполните запрос:

```sql
select sqlite_version();
```

Выйти из `isql`:

```text
quit
```

При успешном подключении должно появиться приглашение `SQL>` и версия SQLite.

## 8. Подключиться из CTRL WinCC OA

Используйте DSN, а не `QSQLITE`:

```cpp
main()
{
  dbConnection conn;
  dbRecordset rs;
  anytype value;
  int rc;

  string connStr = "DSN=WinccSQLite;";
  rc = dbOpenConnection(connStr, conn);

  if (rc != 0)
  {
    DebugN("Ошибка подключения SQLite через ODBC: ", rc);
    return;
  }

  rc = dbOpenRecordset(conn, "SELECT name FROM test", rs);
  if (rc == 0)
  {
    while (!dbEOF(rs))
    {
      if (dbGetField(rs, 0, value) == 0)
        DebugN("name = ", value);

      dbMoveNext(rs);
    }

    dbCloseRecordset(rs);
  }
  else
  {
    DebugN("Ошибка SQL-запроса: ", rc);
  }

  dbCloseConnection(conn);
}
```

## Важные различия между библиотеками

Не указывайте эти файлы в `odbcinst.ini`:

```text
libqsqlite.so
libqsqlodbc.so
```

Это Qt-плагины:

- `libqsqlite.so` используется Qt-приложением через `QSqlDatabase` и драйвер `QSQLITE`;
- `libqsqlodbc.so` — Qt-плагин `QODBC`, который использует unixODBC;
- `/usr/local/lib/libsqlite3odbc.so` — настоящий SQLite ODBC-драйвер для unixODBC.

Рабочая цепочка для CTRL:

```text
CTRL dbOpenConnection()
  -> QODBC3 / libqsqlodbc.so
  -> unixODBC / libodbc.so.2
  -> SQLite ODBC / libsqlite3odbc.so
  -> messages.db
```

## Типовые ошибки

### `Data source name not found`

Проверьте имя DSN и файлы конфигурации:

```bash
odbcinst -j
odbcinst -q -d
grep -A3 '^\[WinccSQLite\]' /etc/odbc.ini
```

### `Can't open lib 'libsqlite3odbc.so'`

Проверьте путь в `/etc/odbcinst.ini`:

```bash
ls -l /usr/local/lib/libsqlite3odbc.so
```

### `QODBC driver not loaded`

Проверьте наличие Qt-плагина `libqsqlodbc.so` в поставке WinCC OA и совместимость его версии Qt с используемым WinCC OA.

### CTRL возвращает ненулевой код

Добавьте диагностику через `dbGetError()` и проверьте права пользователя WinCC OA на файл базы и его каталог.

## Источники

- SQLite ODBC Driver: <http://www.ch-werner.de/sqliteodbc/>
- WinCC OA `dbOpenConnection()`: <https://www.winccoa.com/documentation/WinCCOA/latest/en_US/ControlA_D/dbOpenConnection.html>
- WinCC OA `dbOpenRecordset()`: <https://www.winccoa.com/documentation/WinCCOA/latest/en_US/ControlA_D/dbOpenRecordset.html>
