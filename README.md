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

### Примеры выполнения запросов

Допустим, мы хотим выполнить вызов хранимой процедуры, которая в базе данных была объявлена следующим образом:

```sql
  procedure buy_book(user_id users.id%type, book_id books.id%type, books_count books.available_count%type) is
    currently_available_books books.available_count%type;
    not_enough_books_in_store exception;
    negative_or_zero_count exception;
    begin
      select available_count into currently_available_books from books where id = book_id;

      if currently_available_books - books_count < 0 then
        raise not_enough_books_in_store;
      elsif books_count <= 0 then
        raise negative_or_zero_count;
      end if;
      
      insert into sales values (sales_seq.nextval, sysdate, user_id, book_id, books_count);
      
      exception
        when not_enough_books_in_store then
          raise_application_error(-20001, 'Not enough books in store');
        when negative_or_zero_count then
          raise_application_error(-20002, 'You cannot buy 0 or less books');
    end buy_book;
```

Для этого напишем простую функцию, которая будет принимать параметры, необходимые для вызова процедуры (`user_id`, `book_id`, `books_count`), а возвращать будет массив, состоящий из двух элементов: первый эдемент - строка с запросом, второй элемент - массив параметров. Тут мы используем массив, потому что далее будет удобно пользоватья spread-оператором при вызове `executePLSQL`.

```javascript
const buyBook = (userId, bookId, booksCount) => [
    'begin buy_book(:user_id, :book_id, :books_count); end;',
    [userId, bookId, booksCount]
];
```

Имена параметром в строке запроса не имеют значения, важен только их порядок, который должен совпадать с порядком значений в массиве параметров.

Использовать нашу функцию можно будет следующим образом:

Внутри фукцнии, объявленной как `async`:

```javascript
const f = async() => {
    const plsqlResult = await executePLSQL(...buyBook(1, 1, 15));
    console.log(plsqlResult);
}
```
