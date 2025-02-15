Когда вычислительные узлы Impala и его хранилище данных находятся в разных местах, требования к пропускной способности сети возрастают, так как сетевой трафик включает как получение данных, так и обмен промежуточными результатами.

Чтобы снизить нагрузку на сеть, можно включить кэширование рабочих данных, считываемых с удаленных файловых систем, таких как удаленные узлы HDFS, S3, ABFS, ADLS.

Для включения кэша удаленных данных установите флаг запуска демона Impala
`--data_cache` следующим образом: `--data_cache=dir1,dir2,dir3,...:quota`.

- Флаг устанавливается в виде списка директорий, разделенных запятыми, за которыми следует двоеточие и квота на объем данных в каждой директории.
- Если установить значение пустой строки, кэширование данных будет отключено.
- Кэшированные данные хранятся в указанных директориях.
- Указанные директории должны существовать в локальной файловой системе каждого демона Impala, иначе Impala не сможет запуститься.
- Кроме того, файловая система, в которой находятся директории, должна поддерживать технологию пробивания дыр (hole punching).
- Кэш может использовать до указанных в квоте байтов для каждой из заданных директорий.
- Настройка по умолчанию для `--data_cache` представляет собой пустую строку.

Например, с настройкой, представленной ниже, кэш данных может использовать до 1 ТБ, с максимум 500 ГБ в /data/0 и /data/1 соответственно.
`--data_cache=/data/0,/data/1:500GB`

В Impala версии 3.4 и выше можно настроить одну из следующих политик вытеснения данных из кэша:
- LRU (наименее недавно использованный, по умолчанию)
- LIRS (набор для учета частоты обращений)

LIRS — это устойчивая к сканированию политика с низкими накладными расходами. Настроить политику вытеснения можно с помощью флага
`--data_cache_eviction_policy` при запуске демона Impala:
`--data_cache_eviction_policy=policy`

Примечание: элемент кэша не будет истекать, пока используется та же метаинформация о файле в запросе. Это связано с тем, что ключ кэша состоит из имени файла, времени последнего изменения (mtime) и смещения в файле. Если mtime в метаинформации файла остается неизменным, запрос будет последовательно обращаться к кэшу (при условии достаточной емкости).

### Class `DataCache`
Этот класс представляет собой реализацию кэша данных ввода-вывода, поддерживаемого локальным хранилищем. Он неявно полагается на управление кэшем страниц операционной системы для перемещения данных между памятью и устройством хранения. Это полезно для кэширования данных, считанных с удаленных файловых систем (например, удаленные узлы HDFS, S3, ABFS, ADLS).

Кэш данных разделяется на одну или несколько секций, основанных на строке конфигурации, которая указывает список директорий и соответствующие им квоты на объем хранилища.

Каждая секция имеет кэш метаданных, который отслеживает соответствия ключей кэша и расположения кэшированных данных. Ключ кэша представляет собой кортеж (имя файла, время его последнего изменения, смещение в файле), а запись кэша — это кортеж (файл-хранилище, смещение в файле-хранилище, длина кэшированных данных, необязательная контрольная сумма). Время последнего изменения файла используется для различения версий файла с одинаковым именем. Каждая секция хранит набор кэшированных данных в файлах-хранилищах, созданных на локальном хранилище. При добавлении новых данных в кэш они добавляются в конец текущего используемого файла-хранилища. Потребление хранилища для каждой записи кэша учитывается в квоте этой секции. Когда секция достигает своей емкости, наименее недавно использованные данные вытесняются. Вытесненные данные удаляются из основного хранилища путем пробивания дыр в файле-хранилище, в котором они хранились. Когда файл-хранилище достигает определенного размера (например, 4 ТБ), в него прекращают добавлять новые данные, и вместо этого создается новый файл. Отметим, что благодаря технологии пробивания дыр файл-хранилище на самом деле является разреженным. Например, файл-хранилище может выглядеть следующим образом после нескольких операций вставки и вытеснения. Все пробелы в файле не занимают место на диске.
```
 0                                                           1GB
 +----------+----------+----------+---------+---------+-------+
 |          |          |          |         |         |       |
 |   Data   |   Hole   |   Data   |  Hole   |  Data   |  Hole |
 |          |          |          |         |         |       |
 +----------+----------+----------+---------+---------+-------+
                                                              ^
                                                       insertion offset
```
Опционально, можно включить проверку контрольных сумм, чтобы убедиться, что чтение из кэша соответствует тому, что было вставлено, и что несколько попыток вставки с тем же ключом кэша имеют одно и то же содержимое.

Отметим, что кэш в настоящее время не поддерживает поиск по поддиапазонам и не обрабатывает перекрывающиеся диапазоны. Иными словами, если в кэше есть запись для файла в диапазоне [0,4095], запрос на диапазон [4000,4095] приведет к пропуску, хотя это поддиапазон [0,4095]. Также вставка диапазона [4000,4095] не будет консолидирована с перекрывающимися диапазонами. Иными словами, вставка записей для диапазонов [0,4095] и [4000,4095] приведет к двойному кэшированию данных для диапазона [4000,4095]. На практике это не представляло серьезной проблемы при тестировании с TPC-DS и Parquet, но требует дальнейшего исследования для других форматов файлов и рабочих нагрузок.

Для проверки наличия данных в кэше используется интерфейс `Lookup()`, а для вставки данных в кэш — интерфейс `Store()`. Запись в файл-хранилище и вытеснение из него происходят синхронно. В настоящее время `Store()` ограничивается параллельностью одного потока на секцию, чтобы предотвратить замедление вызывающего процесса в случае, если кэш перегружается и становится ограничен вводом-выводом. Параллельность записи можно настроить с помощью параметра `--data_cache_write_concurrency`. Также `Store()` имеет минимальную гранулярность в 4 КБ, поэтому любые вставленные данные будут округлены до ближайшего кратного 4 КБ.

Количество файлов-хранилищ во всех секциях ограничено параметром `--data_cache_max_opened_files`. После превышения установленного лимита файлы закрываются и удаляются асинхронно потоком в `'file_deleter_pool_'`. Устаревшие записи кэша, ссылающиеся на удаленные файлы, удаляются по мере доступа или косвенно через вытеснение.