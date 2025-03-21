В кафка приходит consumer и забирает нужные сообщения, сдвигая индекс. 
В [[RabbitMQ]] producer пушит consumer.

Producer -> 
	
Cluster ->
	Broker
		Topic
	Broker
		Topic
	Broker
		Topic

Consumer


---

Topic - логическая единица. Она делится на части и каждая чаcтсь называется партицией (partition). В них данные пишутся всегда независимо и в конец партиции. 

Patition - хранится на диске и делится на сегменты (segment). Запись всегда идёт в последний сегмент (активный сегмент).

Активный сегмент - последний сегмент партиции.

Каждый сегмент начинается с базового события (base offset).

Удалять данные можно только сегментами. Если память заканчивается, то удалятся сегменты будут с начала самые старые. 

У каждого сегмента есть 3 характеристики, хранятся в 3 файлах .log .index .timeindex
![[Pasted image 20240801195110.png]]

### Kafka broker
Controller - координирует работу кластера 
![[Pasted image 20240801195330.png]]

Если мы казали что нам нужны 3 копии
![[Pasted image 20240801195420.png]]
Так же В брокеры можно добавлять дополнительные партиции.


На броккерах можно определить партиции лидеры, реплики. из выбранных лидерских партиций данные будут синхронизироваться в реплики, а если лидерская портиция упала, то лидерство берёт следующая партиция, к
![[Pasted image 20240801195633.png]]

Если возвращается выпавшая партиция вернётся в строй, то она должна будет догнаться по данным, но будет уже репликой.
![[Pasted image 20240801195847.png]]

После этого кафка увидит что у нас дисбаланс и автоматически сделает перебалансировку и перевыбор лидера
![[Pasted image 20240801195951.png]]

Выводы:
У каждой партиции есть свой лидер с репликами
Сообщения пишутся в лидера
Данные реплицируются между брокерами
Автоматически фейловер лидерства, за это отвечает контроллер

Дот нет клиент для кафки:
librdkafka

### Producer
Со стороны клиента мы передаём сообщение с ключом и значением
![[Pasted image 20240801200358.png]]

Для раскладывания данных есть специальный enum
![[Pasted image 20240801200539.png]]

Как продюсить сообщения в кафку
Первый способ
![[Pasted image 20240801200724.png]]

Лучше делать вот так
![[Pasted image 20240801200755.png]]
![[Pasted image 20240801200811.png]]

Второй способ
![[Pasted image 20240801200846.png]]
При этом будет вызван callback, когда будет доставлено это событие
![[Pasted image 20240801200950.png]]

#### Гарантии доставки
![[Pasted image 20240801201041.png]]
1. Никаких гарантий
2. Доставка до лидера
3. Доставка в лидера и реплики
	1. Если в процессе одна из реплик упала, то продюссер всё равно будет ждать, когда во всех реплики запишется сообщение. Чтобы такого избежать, есть настройка, min.insync.replicas = 2, указывающая на минимально количество реплик, в которое нужно записать ![[Pasted image 20240801201559.png]]


#### Производительность
1. Низкая задержка
2. Высокая пропускная способность

Данные в брокер отправляются пачками. Для настройки "Как это делать", есть 3 параметра:
BatchSize - размер сообщений
BarchNumMessages - количество сообщений
LingerMs - время, через которое будет отправлены сообщения независимо от размера и количества

Так же есть сжатие
![[Pasted image 20240801202229.png]]

**Выводы:**
Producer выбирает партицию для отправки сообщения
Producer определяет уровень гарантии доставки
В Producer можно тюнить производительность

### Consumer
![[Pasted image 20240801202530.png]]

Может читать данные пачками, запоминает индекс для следующего сообщения и читает следующую пачку
![[Pasted image 20240801202712.png]]

Если мы подписаны на несколько партиций, то можем читать данные сразу с нескольких . Но не гарантируется порядок чтения
![[Pasted image 20240801202811.png]]

Если вдруг что-то слетело, и индекс след. сообщения не сохранился, то нам придут заново те же данные. Чтобы этого избежать стоит с начала процессить данные, а потом коммитить успешный результат
![[Pasted image 20240801204109.png]]

Но так делать не очень эффективно. Лучше обрабатывать пачками и отправлять событие на успешный коммит не на каждое сообщение, а на пачки сообщений.

Более предпочтительное использование, в котором можно настроить автоматиеские коммиты и время, за которое они будут происходить
![[Pasted image 20240801204518.png]]

Как понять откуда начинать читать, если мы делаем это впервые? Варианта всего 3:
1. Latest - Всё что есть в топике нас не волнует, мы начнём с нового
2. Earliest - Отматываем в самую древность и начинаем читать оттуда
3. Error - Если вдруг пропали offset, мы случайно подтёрли данные

![[Pasted image 20240801204746.png]]

Consumer Group
Если у нас есть несколько брокеров, которые расположены на разных серверах, то по умолчанию comsuner будет читать из leader, но если comsumer расположен на другой машине, то для производительности можно настроить на чтение данных из реплики, чтобы не не гонять лишний раз данные по сети. Это можно реализовать настроив ClientRank
![[Pasted image 20240801205607.png]]
![[Pasted image 20240801205645.png]]

**Выводы:**
1. "Smart" Consumer, между собой договариваются как и что будут читать
2. Consumer поллит кафку, запрашивает пачку, клиент отдаёт по 1 событию
3. Consumer отвечает за гарантию обработки
4. Автоматически фейловер в Consumer-группе, при падении одного из Consumer
5. Независимая обработка разными Consumer-группами


**Преимущества, которые даёт Kafka:**
- Персистентность данных, данные хранятся на диске (а у RabbitMq в памяти)
- Высокая производительность
- Независимость пейплайнов обработки
- Возможность прочитать историю сообщений заново
- Гибкость в использовании, большое количество настроек

**В Kafka нет:**
- Отложенных сообщений
- DLQ
- AMQP | MQTT
- TLL На сообщения
- Очереди с приоритетами