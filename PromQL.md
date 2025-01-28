rate 
increase



Колличество ошибок в минут
sum(increase(flask_app_requests_total {job="test", http_status=~"4.+|5.+"}[1m])) by (http_status)

Коллиество запрросов к приложению в минуту со всех подов, и разделение их по статусу, для понимаю какие статутос какое колличество 
sum(rate(flask_app_requests_total {job="test"}[1m])) by (http_status)


sum(increase(flask_app_requests_total {job="test", http_status=~"2.+|3.+"}[1m])) / sum(increase(flask_app_requests_total {job="test"}[1m]))
Процент успешных запросов общего числа в минуту, должно быть близко к 100 процентам
