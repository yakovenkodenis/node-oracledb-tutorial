# Инструкция по работе с Oracle Database XE в среде node.js

------------------

## Требования к программному обеспечению

1. node.js v7+
2. npm v3+
3. Oracle Database 11g XE

Данный гайд ориентирован на работу в среде Unix, но, наверное, может сработать и на Windows. А может и нет.


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

Вызов `oracledb.getConnection` вернёт промис, в который будет передан объект `connection`. Объект `connection` позволяет выполнять запросы непосредственно к базе данных (метод `connection.execute`). После выполнения необходимых операций с базой данных, соединение `connection` нужно освобождать, используя метод `connection.release()`.

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

Для этого напишем простую функцию, которая будет принимать параметры, необходимые для вызова процедуры (`user_id`, `book_id`, `books_count`), а возвращать будет массив, состоящий из двух элементов: первый элемент - строка с запросом, второй элемент - массив параметров. Тут мы используем массив, потому что далее будет удобно пользоватья spread-оператором при вызове `executePLSQL`.

```javascript
const buyBook = (userId, bookId, booksCount) => [
    'begin buy_book(:user_id, :book_id, :books_count); end;',
    [userId, bookId, booksCount]
];
```

Имена параметров в строке запроса не имеют значения, важен только их порядок, который должен совпадать с порядком значений в массиве параметров.

Использовать нашу функцию можно будет следующим образом:

Внутри фукцнии, объявленной как `async`:

```javascript
const f = async() => {
    const plsqlResult = await executePLSQL(...buyBook(1, 1, 15));
    console.log(plsqlResult);
}
```

Используя `.then` на объекте промиса, возвращаемого функцией `executePLSQL`:

```javascript
executePLSQL(...buyBook(1, 1, 15))
  .then(data => console.log(data))
  .catch(err => console.log(err));
```

Ниже представлен пример выполнения хранимой в бд функции, которая возвращает значение.

Пример кода функции в бд:

```sql
  function get_books_csv_by_publisher(publisher_id in publishers.id%type) return varchar2 as
    result varchar2(2000);
    book_name books.name%type;
    cursor books_cur is select books.name from books where books.publisher_id = publisher_id;
    begin
        result := '';
        open books_cur;
        loop
          fetch books_cur into book_name;
          exit when books_cur%notfound;
          result := result || book_name || ',';
        end loop;
        close books_cur;
        
        return result;
    end get_books_csv_by_publisher;
```

В случае, когда нам нужно получить значение, возвращаемое хранимой в бд функцией, `connection.execute` вторым параметром будет принимать объект специального вида, а не массив, как раньше. Объект должен будет содержать поля, обозначающие тип возвращаемого значения, его размер и направление (н-р, BIND_OUT). 

Для этого напишем вспомогательную функцию, которая будет принимать массив наших параметров для запроса и тип возвращаемого значения, а возвращать будет объект нужного вида. `zipObject` - функция из пакета [lodash](https://github.com/lodash/lodash).

```javascript
const zipObject = require('lodash/zipObject');

const zipParams = (params, type) => {
    Object.assign({}, zipObject([...Array(params.length).keys()], params))
    return Object.assign({}, zipObject([...Array(params.length).keys()], params), {
        result: { dir: oracledb.BIND_OUT, type, maxSize: 2000 }
    });
};
```

Теперь напишем вспомогательную функцию `getBooksCSVbyPublisher` по типу ранее описанной `buyBook`:

```javascript
const getBooksCSVbyPublisher = (id) => [
    `begin :result := ${PACKAGE}.get_books_csv_by_publisher(:0); end;`,
    zipParams([id], oracledb.STRING)
];
```

Примеры использования представлены ниже.

Внутри `async` функции:

```javascript
const f = async() => {
    try {
        const dbResponse = await executePLSQL(...getBooksCSVbyPublisher(1));
        const jsonResponse = JSON.stringify(dbResponse.outBinds.result.slice(0, -1));
        console.log(jsonResponse);
    } catch (e) {
        console.log(e);
    }
}
```

Второй вариант:

```javascript
executePLSQL(...getBooksCSVbyPublisher(1))
  .then(dbResponse => {
        const jsonResponse = JSON.stringify(dbResponse.outBinds.result.slice(0, -1));
        console.log(jsonResponse);
  })
  .catch(err => console.log(err));
```

-------------------------

Выше были описаны базовые примеры использования драйвера [node-oracledb](https://github.com/oracle/node-oracledb). Для более продвинутых вариантов использования стоит обращаться к [официальной документации](https://github.com/oracle/node-oracledb/blob/master/doc/api.md)

----------------------
