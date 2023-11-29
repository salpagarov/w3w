# w3w
## Структуры
### Список известных узлов

Нужен для "холодного старта" сети в фазе INIT.

1. Сохраняется между сеансами.
2. Пополняется в процессе работы.
3. Сортируется по убыванию последнего времени взаимодействия.

~~~json
{ "<ip>" : "<timestamp>", ... }
~~~

### Кольцо маршрутизации

1. Элементы алфавитно отсортированы по возрастанию.
2. У каждого элемента есть левый (младший) и правый (старший) элементы.
2. Изначально (в фазе INIT) пустое.

~~~json
[
    {
        "<address>" : {
            "ip" : "<ip>",
            "port" : "<port>",
            "direction" : "<in|out>"
        }, ...
    }
]
~~~

### Идентификатор узла

Адрес в сети W3W. Создается в фазе INIT. Рабочий вариант `NID := blake2(UUID)`

## Сообщения

### HERE

Содержание: "version HERE nid @ ip:port". Отправляется на 22000/udp.

- version - версия протокола (два символа шестнадцатеричного представления одного байта) с префиксом w. Текущее значение `w00`.
- nid - идентификатор узла.
- ip - сетевой адрес (пока только IPv4).
- port - при отправке сообщения исходящий и целевой порты должны быть одинаковы.

### MESSAGE

Структура:

- версия "wXX"
- отправитель (NID)
- получатель (NID)
- произвольное двоичное содержимое (DATA)

## Логика

### Кольцо маршрутизации

| State       | Description                               | Condition                | TRUE    | FALSE  |
| ----------- | ----------------------------------------- | ------------------------ | ------- | ------ |
| NEW_NODE id | pointer := 0                              | TRUE                     | NEXT    |        |
| DEL_NODE id | Удалить элемент с nid.ident == id         | TRUE                     | RE_SORT |        |
| NEXT        | ++pointer                                 | pointer < count(ring)    |         | RESET  |
|             |                                           | is_exists(ring[pointer]) | COMPARE | INSERT |
| COMPARE     |                                           | nid > ring[pointer]      | NEXT    |        |
| RESET       | pointer := 0                              | TRUE                     | NEXT    |        |
| INSERT      | ring[pointer + .5] = nid                  |                          |         |        |
| RE_SORT     | Привести все индексы к натуральным числам | TRUE                     | STOP    |        |
| STOP        |                                           |                          |         |        |


### Сетевой модуль

|   State   | Description                                                     |       Condition        |   TRUE    |   FALSE   |
| :-------: | :-------------------------------------------------------------- | :--------------------: | :-------: | :-------: |
|   INIT    | Прочесть список известных узлов в nodes                         |                        |           |           |
|           | Пустое кольцо маршрутизации ring := []                          |                        |           |           |
|           | myNID := hash(UUID)                                             |                        |           |           |
|           | Определить myIP                                                 |          TRUE          |   NONCE   |           |
|  NOUNCE   | port := 22000                                                   |                        |           |           |
|           | Отправить каждому из nodes сигнал "HERE" с ++port               |          TRUE          | RECONNECT |           |
| RECONNECT | Установить отсутствующие подключения для ring @ direction = out | Соединение установлено |           | DEL_ROUTE |
|   WAIT    | Проверить 22000/udp                                             |         "HERE"         |  SIGNAL   |           |
|           | Проверить 22000/tcp                                             |  Входящее соединение   | ADD_ROUTE |           |
|           | Проверить все установленные соединения                          |       "MESSAGE"        |  RECEIVE  |           |
|           | Проверить очередь на отправку                                   |       "MESSAGE"        |   SEND    |           |
|           | Ожидание                                                        |          TRUE          |   WAIT    |           |
|  SIGNAL   | Добавить node в ring @ direction = out                          |    size(ring) > max    | OPTIMIZE  | RECONNECT |
| OPTIMIZE  | Найти в ring три узла, которые "слишком близко"                 |                        |           |           |
|           | Отправить двум крайним HERE с данными среднего                  |                        |           |           |
|           | Убрать средний узел из ring и оборвать с ним соединение         |          TRUE          | RECONNECT |           |
| ADD_ROUTE | Добавить node в ring @ direction = in                           |    size(ring) > max    | DEL_ROUTE | RECONNECT |
| DEL_ROUTE | Убрать node in ring                                             |          TRUE          | RECONNECT |           |
|  RECEIVE  | Проверить получателя                                            |  MESSAGE.dest = myNID  |   SAVE    |   SEND    |
|   SEND    | Найти старший и младший узел для MESSAGE.dest                   |                        |           |           |
|           | Отправить MESSAGE им                                            |        Успешно         |           | DEL_ROUTE |
|           | Удалить MESSAGE из очереди                                      |          TRUE          |   SAVE    |           |
|   SAVE    | Сохраняем пакет                                                 |          TRUE          |   WAIT    |           |

### Сохранение сообщения

В базу данных KV:

- key: HASH(DATA)
- value: DATA

Отправляемые блоки также сохраняются в базе
