# Инструкция по работе с Oracle Database XE в среде node.js

------------------

## Требования к программному обеспечению

1. node.js v7+
2. npm v3+
3. Oracle Database 11g XE

Данный гайд ориентирован на работу в среде Unix, но, наверное, можен сработать и на Windows. А может и нет.


### Устанавливаем драйвер базы данных (версия 1.11.0)

```bash
npm install --save oracledb@1.11.0
```

-----------------

## Подключение к Oracle DB

### Функция подключения к базе данных и выполнения запроса (Promise-based):

```javascript
const oracledb = require('oracledb');

const executePLSQL = (statement, params=[]) =>
    new Promise((resolve, reject) => {
        oracledb.getConnection(
            {
                user: process.env.DB_USER,
                password: process.env.DB_PASSWORD,
                connectString: process.env.DB_CONNECTION_STRING
            }
        ).then(connection => {
            return connection.execute(
                statement, params
            ).then(result => {
                resolve(result);
                return connection.release();
            }).catch(err => {
                reject(err);
                return connection.release();
            })
        }).catch(err => reject(err));
    });
```

Использование промисов позволит в дальнейшем использовать синтаксис `async/await` для выполнения запросов. Однако, [официальный драйвер](https://github.com/oracle/node-oracledb) предоставляет не только Promise-based интерфейс, но и callback-based.

Описанная выше функция принимает параметры `statement` и `params`. Параметр `statement` - PL/SQL-запрос, а `params` - параметры, которые необходимо передать в запрос (об этом ниже).

Метод `getConnection` на объекте `oracledb` принимает в качестве параметра объект с конфигурациями подключения (имя пользователя, пароль и адрес для подключения к базе данных). Сами параметры лучше хранить в переменных среды (например, использовать пакет [dotenv](https://github.com/motdotla/dotenv)) и потом получать доступ к ним через `process.env.$PARAM_NAME`, как показано в коде выше.

Пример параметра `connectString` для Oracle DB XE, установленной локально со всеми параметрами по умолчанию: `localhost/XE`.

Выозов `oracledb.getConnection` вернёт промис, в который будет передан объект `connection`. Объект `connection` позволяет выполнять запросы непосредственно к базе данных (метод `connection.execute`). После выполнения необходимых операций с базой данных, соединение `connection` нужно освобождать, используя метод `connection.release()`.

