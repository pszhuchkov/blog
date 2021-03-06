---
layout: post
title:  "Деструктивный синтаксис в функциях"
permalink: /destructuring/
tags: python fp destructuring syntax
---

С удивлением обнаружил, что в Питоне редко пользуются деструктивным
синтаксисом в сигнатурах функций. Рассмотрим, какие преимущества он
дает.

Деструктивный синтаксис пришел, как все лучшее, из мира
функционального программирования. В Питоне он известен как "распаковка
кортежа" на составные переменные.

Распаковывать можно не только кортеж, но и списки и вообще все
итерируемые коллекции. Правда, для множеств и словарей не определен
порядок итерации, что может привести к багам.

Простой пример:

~~~ python
point = (1, 2)
x, y = point
~~~

Данные часто собираются в группы. Например, точка -- это два числа,
которые удобно хранить и передавать вместе. Результат функции тоже
может быть парой -- флаг успеха и результат. В кортежи удобно собирать
права доступа, табличные данные.

Некоторые программисты теряются на этом месте и пишут классы, чтобы
группировать данные. Переход к классам уводит нас в противоположную
сторону от коллекций и всех их преимуществ. Лучше хранить данные в
коллекциях насколько это возможно.

Например, работаем с точками. Есть две точки, нужно вычислить
декартово расстояние. Решение в лоб:

~~~ python
point1 = (1, 2)
point2 = (3, 4)

def distance(p1, p2):
    x1, y1 = p1
    x2, y2 = p2
    ...

print distance(p1, p2)
~~~

Хорошо, но лишние строки на распаковку. Решение в ООП-стиле:

~~~ python
class Point(object):

    def __init__(self, x, y):
        self.x = x
        self.y = y

    def distance(self, other):
        # use self.x, self.y, other.x, other.y
        # to refer variables
        ...

p1 = Point(1, 2)
p2 = Point(3, 4)
print p1.distance(p2)
~~~

Лишние строки на класс. К тому же, я забыл добавить метод `__str__` и
во время дебага увижу что-то вроде `<object Point at 0x523fw523>`
вместо чисел. Матерясь, полезу дописывать метод и только увеличу
энтропию.

Самая простая, и потому лучшая реализация:

~~~ python
point1 = (1, 2)
point2 = (3, 4)

def distance((x1, y1), (x2, y2)):
    ...

print distance(p1, p2)
~~~

Видим, что распаковка происходит на уровне сигнатуры. Это значит, в
теле функции уже доступны компоненты точек и остается только посчитать
значение.

Такая функция лучше защищена от неправильных аргументов. Если передать
кортеж не с двумя, а тремя компонентами, ошибка распаковки случится
раньше, еще до входа в функцию.

Деструктивный синтаксис следует принципу "явное лучше неявного". Он
избавляет от долгих описаний вроде "аргумент `permission` -- это
кортеж, где первый элемент то, второй се, третий...". Сигнатура с
распаковкой скажет сама за себя.

Распаковывать кортежи можно и в лямбдах. Я пользуюсь этим для
обработки словарей, когда нужна пара ключ-значение:

~~~ python
data = {'foo': 1, 'bar': 2, 'baz': 3}
process = lambda (key, val): 'key: %s, value: %d' % (key, val)
print map(process, data.iteritems())
>>> ['key: baz, value: 3', 'key: foo, value: 1', 'key: bar, value: 2']
~~~

Деструктивный синтаксис сокращает код, делает его декларативным, а
значит, удобным в сопровождении.
