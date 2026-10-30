**1. Функция для получения датафрейма из таблицы в hadoop**

```python
from impala.dbapi import connect

from datetime import datetime

import pandas as pd

​

def log_msg(*args):

    '''

    Логирование сообщений с записью даты и времени

    '''

    msg = '. '.join([str(arg) for arg in [*args]])

    print(datetime.now(), f': {msg}', sep='')

   

    

    

    

​

def get_df_from_hadoop(query, db_connection):

    '''

    Выполнение запроса к базе данных для получения датафрейма из таблицы в БД

    '''

    try:

       

        conn = db_connection

        cursor = conn.cursor()

        sql = query

        try:

            cursor.execute(sql)

            df = pd.DataFrame(cursor.fetchall(), columns = [desc[0] for desc in cursor.description])      

        except Exception as error:

            log_msg(f"Ошибка запроса: {error}")

           

        finally:

            cursor.close()

            conn.close()

           

    except Exception as error:

        log_msg(f"Ошибка подключения: {error}")

        return None         

    return df

​

hadoop_connection = connect(host='hdp-p3pml', port=21050,  use_ssl='true', auth_mechanism='GSSAPI')

sql_query = f"""

            select *

            from table

            """

​

df = get_df_from_hadoop(sql_query, hadoop_connection)
```
