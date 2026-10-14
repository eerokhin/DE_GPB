**Подключения**

```python
import os
import getpass
import traceback
import pandas as pd
from impala.dbapi import connect
import re
import time

def impala_connect():
  return connect(host='hdp-p8pml', port=22222, use_ssl='true', auth_mechanism='GGAPI')

login = getpass.getuser()
print('Введите пароль')
password = getpass.getpass()

# Получение тикета доступа для подключения к БД
os.system('echo ' + password + ' | kinit ' + login)
!klist

conn = impala_connect
cursor = conn.cursor()
```

**1.Скрипт для создания таблиц с нуля. Дроп-креэйт. С возможностью задать таблицы которые не трогаю.**

```python
##1.Скрипт для создания таблиц с нуля. Дроп-креэйт. С возможностью задать таблицы которые не трогаю.

#Список таблиц которые не трогаю (не удаляю и не создаю)
tables_to_keep = {'schema.table' }

#with open("/work /1_create_table.sql", "r", encoding = 'cp1251') as f:
#    sql = f.read()

with open("/work/drop_create_table/1_create_table.sql", "r", encoding = 'UTF8') as f:
    sql = f.read()

   

#Удаляю комментарии   
sql = re.sub(r'/\*.*?\*/', '', sql, flags = re.DOTALL) 
sql = re.sub(r'--.*', '', sql)
queries = sql.split("/")
for i, query in enumerate(queries):
    query = query.strip()  
    if not query:
        continue
    drop_match = re.match(r'DROP TABLE IF EXISTS\s+([^\s;]+)', query, flags=re.IGNORECASE)   
    if drop_match:
        table_name = drop_match.group(1).strip('`')
        if table_name in tables_to_keep:
            print(f"---Skip DROP TABLE {table_name} ---")
            continue   
    create_match = re.match(r'CREATE TABLE\s+IF NOT EXISTS\s+([^\s(]+)', query, flags=re.IGNORECASE)    
    if create_match:
        table_name = create_match.group(1).strip('`')
        if table_name in tables_to_keep:
            print(f"---Skip CREATE TABLE {table_name} ---")
            continue  
    try:
        print(f"\n---Runnig query {i}---")
        print(query[:100])
        cursor.execute(query)
        time.sleep(1)
    except Exception as e:
        print(f"Error in query {i}: {e}")
       
print('---------------Создание завершено---------------')        
cursor.close()
conn.close()
```

**2.Скрипт для удаления таблиц без создания. С возможностью задать таблицы которые не трогаю.**

```python
##2.Скрипт для удаления таблиц без создания. С возможностью задать таблицы которые не трогаю.
#Список таблиц которые не трогаю (не удаляю и не создаю)

tables_to_keep = {                
                  'schema.table'
                  }

#with open("/work /1_create_table.sql", "r", encoding = 'cp1251') as f:
#    sql = f.read()

with open("/work /1_create_table.sql", "r", encoding = 'UTF8') as f:
    sql = f.read()

   
#Удаляю комментарии   
sql = re.sub(r'/\*.*?\*/', '', sql, flags = re.DOTALL) 
sql = re.sub(r'--.*', '', sql)
queries = sql.split("/")
for i, query in enumerate(queries):
    query = query.strip()  
    if not query:
        continue

    drop_match = re.match(r'DROP TABLE IF EXISTS\s+([^\s;]+)', query, flags=re.IGNORECASE)   
    if drop_match:
        table_name = drop_match.group(1).strip('`')
        if table_name in tables_to_keep:
            print(f"---Skip DROP TABLE {table_name} ---")
            continue     
        #Удаляю таблицы
        try:
            print(f"\n---Runnig query, drop table {table_name}---")
            print(query[:100])
            cursor.execute(query)
           time.sleep(1)
        except Exception as e:
            print(f"Error in query {i}: {e}")
print('---------------Очистка завершена---------------')       
cursor.close()
conn.close()
```

**5.Скрипт для выполнения запросов на вставку INSERT.**

```python
## 5.Скрипт для выполнения запросов на вставку INSERT. Новая версия с помощью библиотеки sqlparse.
## Предыдущая реализация выдавала ошибку если в select было деление / и тд.
#Запросы выполняюются последовательно, друг за другом
import sqlparse
import re
import time

#with open("/work/ 3_update_table.sql", "r", encoding='cp1251') as f:
#    sql = f.read()

with open("/work/ 3_update_table.sql", "r", encoding='UTF8') as f:
    sql = f.read()

# 1. Удаление комментариев
sql = sqlparse.format(sql, strip_comments=True)

# 2. Замена ТОЛЬКО разделителей "/" (которые стоят отдельной строкой) на ";"
sql = re.sub(r'^\s*/\s*$', ';', sql, flags=re.MULTILINE)

# 3. Разбиение на запросы
queries = sqlparse.split(sql)

# 4. Увеличиваем лимит памяти для запросов текущей сессии
#cursor.execute("set mem_limit = 10g")

total_start = time.time()
stats = []

for i, query in enumerate(queries):
    query = query.strip()
    if not query:
        continue
    insert_match = re.search(r'INSERT\s+INTO\s+([^\s(]+)', query, flags=re.IGNORECASE)
    stats_match = re.search(r'COMPUTE\s+STATS\s+([^\s;]+)', query, flags=re.IGNORECASE)
    if insert_match:
        table_name = insert_match.group(1).strip('`')
        query_type = "INSERT"
    elif stats_match:
        table_name = stats_match.group(1).strip('`')
        query_type = "STATS"
    else:
        continue

    print(f"\n--- [{i}] {query_type} - {table_name} ---")

    start_time = time.time()

    try:
        cursor.execute(query)
        duration = time.time() - start_time
        print(f"Выполнено {duration:.2f} sec")
        stats.append((table_name, query_type, duration, "OK"))
    except Exception as e:
        duration = time.time() - start_time
        print(f"Error after {duration:.2f} sec: {e}")
        stats.append((table_name, query_type, duration, "ERROR"))
total_time = time.time() - total_start

cursor.close()
conn.close()
print('---------------Статистика---------------')

for table, qtype, duration, status in stats:
    print(f"{status:6} | {qtype:6} | {duration:8.2f} sec | {table}")

print(f"\nTOTAL TIME: {total_time:.2f} sec")
```
