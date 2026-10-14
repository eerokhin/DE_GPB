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
