# Sberbank
See https://contest.sdsj.ru/

## Подготовка: ##

Нам понадобятся следующие библиотеки:
- xgboost
- numpy
- pandas
- sklearn
- statsmodels

Также предобработаем исходные файлы:
- Удалим header из файлов событий и трейна
- Переименнуем файл customers_gender_train.csv в train, а transactions.csv в events
- Сформируем файл test из customerId, которые отсутствуют в train, но присутствуют в events
- Запустим скрипт prepare.sh, который посчитает различные словари для задачи А
```bash
bash prepare.sh events train
```

## Задача A ##

#### Как запустить: ####

```bash
cd A
bash run.sh 25 ../train ../test ../events
```

#### Как это работает: ####

25 - это количество моделей, которые будут обучены для усреднения предсказания. Дожидаться такого большого количества нет смысла, поскольку это практически не улучшает решение.
Основной алгоритм решения - генерируем фичи, обучаем xgboost в режиме классификации, предсказываем.

#### Основные фичи: ####
- элементарные фичи вида "количество событий для customer", "сумма расходных транзакций"
- bag of words для самых популярных терминалов
- статистика трат для mcc/type/дня недели/дня месяца/месяца/часа/типа дня/пары последовательных mcc
- статистика суммарных трат и заработков по особым "мужским" и "женским" mcc/type. Особые mcc/type выбраны исходя из суммарных и средних трат мужчин и женщин в соответствующей категории(код выбора находится в prepare.sh)
- наивный Байес, обученный как мета-фича на 0.9 части learn'а и применена к остальной 0.1 части
- также качество увеличилось если игнорировать автоматические транзакции(две транзакции подряд в один момент с суммой 0)


## Задача B ##

#### Как запустить ####
```bash
cd B
python gauss.py ../events 457 487 submission
```

#### Как работает ####

Основной алгоритм - используем multi-output gaussian process. В качестве X выступают фичи зависящие от текущего дня и фичи для каждого MCC, все вместе - один вектор из 2400 чисел. В качестве Y - 184 ответа для текущего дня для всех MCC. Не берем в обучение первые 3 месяца чтобы фичи были посчитаны везде одинаково.

#### Основные фичи ####

- день недели
- для последних X месяцев считаем средние(логарифмов) траты в день/средние траты в этот день недели/средние траты в выходные и будни для X in [1..3]
- среднее логарифмов трат за все время для дней и месяцев для всех дней и только будни/выходные 
- ядро для gaussian process было выбрано полу-случайным способом на основе результатов на валидейте и равно взвешенной сумме ядер RBF, White, RationalQuadratic и DotProduct

#### Стоит отметить ####

В процессе решения было испробовано по меньшей мере 5 подходов к решению задачи. Также очень хороший прирост давало смешение предсказание различных методов. Приведенный здесь подход Не давал наилучшего результата на public leaderboard(а лишь 1.658 вместо лучших 1.636), а был выбран случайно как последняя попытка =). Какие подходы были:
- сделать xgboost для всех mcc вместе. Давало неплохое качество, порядка 1.655 на public'е
- сделать xgboost для каждого mcc отдельно. На отдельных mcc очень выигрывало у прошлого подхода, в виде смеси давало порядка 1.65 
- сделать ARIMA с 'exog' фичами описанными выше только для 1 прошлого месяца. В смеси с предыдущими методами выдавало результат порядка 1.645 
- сделать one-output gaussian process для каждого mcc по отдельности. Давало 1.644 как отдельная модель. Смешанная с ARIMA давало порядка 1.636

## Задача C ##

#### Как запустить ####
```bash
# Приготовим фичи для customer'ов при помощи кода из задачи А
cd A
python main.py -D -l ../train -v ../test -E ../events -s ../stat/ > ../C/user_feature
# Запустим само решение
cd ../C
python main.py -E ../events -R 0 -T 15500 -O submission -d 5 -e 0.001 -P -v ../test -V
```

#### Как работает ####

Основной алгоритм - генерируем фичи про прошлым месяцам, обучаемся только на последнем месяце, минимизируем MSE, делаем миллионы итераций средне-глубоких деревьев, очень долго ждем, предсказываем.

#### Основные фичи ####

- фичи про пользователя из задачи А
- логарифм средних траты человека в этой категории в последние X дней для X in [1..N]
- для каждого из предыдущих месяцев считаем сумму трат, количество трат, сумму трат нормированную на число дней, заработок человека, траты человека, общие траты в этой категории
- за все прошлое время суммарные траты, суммарное количество трат, предсказанное количество трат за всю историю нормированное на число дней
