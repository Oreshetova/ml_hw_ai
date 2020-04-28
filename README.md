# Домашнее задание по курсу "Машинное обучение ч.2" тема AI

Предлагается реализовать агента для простой многопользовательской игры с другими агентами.
Это вариант многораундовой ультимативной игры.

## Правила игры

Для игры выбирают двух случайных участников. Дальше выбирают в паре участника (proposer), 
которому дают возможность сделать ультиматум. Наприме есть 100 долларов/очков/леденцов. 
Он (proposer) должен разделить эти плюшки в каком-то соотношении. К примеру, 100 себе и 0 второму участнику (responder). 
Или 70 себе и 30 второму участнику. Второй участник (responder) может либо согласиться на этот ультиматум.
Тогда произойдёт сделка, каждый получит в соответствии с предложенным разбиением. 
 Либо отказаться. Тогда оба получают 0.

Проводится множество раундов, после которых сравниваются статистики набранных долларов/очков/леденцов 
и распределяются места. Набравшему болбьше всех первое и т.д.

## Метрика успеха

В качестве метрики успешности агента принимается нижняя граница доверительного интервала в две сигмы 
от его среднего заработка.
Если `gains` - это массив заработков размера `n`, то `score = mean(gains) - 2 * std(gains) / sqrt(n)`. 

## Использование песочницы
 <_Пока в разработке следите за новостями_>

## Создание своего агента

### Структура класса
Для написания своего агента, необходимо создать класс, отнаследовавшись от базового класса 
`agent.base.BaseAgent`. Далее нужно реализовать следующие методы:
- `def get_my_name(self)` возвращяет строкой ваше имя и фамилию 
- `def offer_action(self, data)` возващяет размер вашего предложения (int) некоторому агенту в раунде. 
Данные об агенте и общем размере разделяемых средства находятся в `data` 
структура класса описана здесь `base.protocol.OfferRequest`. Размером вашего предложения `offer` это то сколько 
вы предлагаете оставить другому агенту, в случае принятия предложения, ваша награда будет равна `total_amount - offer`.
- `deal_action(self, data)` возвращяет ответ (bool) согласны ли вы на предложение 
от некоторого агента. Данные об агенте и о размере предложения находятся в `data` 
структура класса описана здесь `base.protocol.DealRequest` 
- `def round_result_action(self, data: RoundResult)` метод вызывается по окончанию раунда, собщяет общие 
результаты в `data` описание `base.protocol.RoundResult`. 

### Структуры данных и их поля
#### OfferRequest
```
round_id: int - номер раунда
target_agent_uid: str - идентификатор агента которому будет отосланно предложение
total_amount: int - общее количество которое необходимо разделить
```
#### DealRequest
```
round_id: int - номер раунда
from_agent_uid: str - идентификатор агента от которого поступило предложение
total_amount: int - общий размер разделяемых средств
offer: int - предложение от агента
```
#### RoundResult
```
round_id: int - номер раунда
win: bool - True если раунд успешен и предложение принято, False - в противном случае
agent_gain: Dict[str, int] - ассоциативный массив с размерами наград, ключ - идентификатор агентов раунда
disconnection_failure: bool - флаг показывает, что во время раунда произошел дисконект одного из участников
```

### Пример агента
Простой агент без памяти, который всегда делит поровну и принимает любое не нулевое предложение.
```python
class DummyAgent(BaseAgent):

    def get_my_name(self):
        return 'Dummy'

    def offer_action(self, m):
        return m.total_amount // 2

    def deal_action(self, m):
        if m.offer > 0:
            return True
        else:
            return False

    def round_result_action(self, data):
        pass
```

### Общий алгоритм использования агента в песочнице
1. Клиент создается ваш агент через `__init__` после создания отсылает серверу сигнал готовности.
2. Сервер дожыдается некоторое время участников и начинает серию раундов.
3. Случайным образом выбираются пары агентов, один proposer другой responder.
4. Сервер спрашивает у proposer размер его предложения (вызывается метод offer_action) и отсылает результат
к responder. 
5. У responder вызывается метод deal_action и получив ответ он отсылается на сервер который решает исход раунда.
6. В случае окончания раунда либо дисконекта одного из участников, сервер отсылает результат агентам и у них вызывается round_result_action.
7. Это повторяется много. раз пука не наберется достаточная статистика по результатам игр агентов. 
8. Классы агентов все это время находятся в памяти клиентов и могу хранить состояние предыдущих раундов.

### Общие правила, рекомендации и советы
1. Запрещяется пользоваться файловой системой (читать и писать файлы) и делать запросы в интернет.
2. Весь код агента должен находится в томже файле где и класс агента. Разрешается использовать: 
стандартные баблиотеки python 3.7, numpy, scikit-learn. Если потребуются другие, напишите в slack, при необходимости добавим.
3. Метода `get_my_name` должен возвращять ваше имя и фамилию.
4. В качестве идентификаторов агента используются строковые представления UUID. После того как подключатся все агенты,
 сервер оповестит вас о вашем uid, его можно будет узнать из поля agent_id. Обратите внимание, после инициализации класся,
 вызова `__init__`,  это поле будет `None` до оповещения от сервера. Однако гарантируется что `agent_id` будет доступен 
 когда будут вызываться методы `offer_action`, `deal_action` и `round_result_action`.
5. Если агент не отвечает более чем 2 секунды, например долгий цикл обработки в `offer_action`, он удаляется из обработки, 
никакие дальнейшие его сообщения не обрабатываются. 
6. Если нужна длительная инициализация, проведите ее в методе `__init__`, так как сервер будет 1 минуту ждать инициализации
всех агентов.
7. Агент должен не упасть с ошибкой или не дисконектнуться хотябы в 1% раундов чтобы его очки учитывались.
8. За любые попытки прямого использования сетевого протокола игры и прочие хаки системы, будет засчитан провал задания. 

Следите за обновлениями новостей на сайте и в slack, могут быть скорректированны правила или добавдленны утилиты для повышения удобстав.