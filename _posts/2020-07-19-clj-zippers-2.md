---
layout: post
title:  "Зипперы в Clojure (часть 2). Автонавигация"
permalink: /clj-zippers-2/
tags: clojure zippers
---

{% include zipper-toc.md %}

Итак, мы разобрались с тем, как перемещаться по коллекции. Однако у читателя
возникнет вопрос: откуда приходит путь? Как мы узнаем заранее, в каком
направлении двигаться?

Выскажем тезис, который по праву считается главным во всей главе. **Ручная
навигация по данным лишена всякого смысла.** Если путь известен заранее, вам не
следует использовать зиппер — это лишнее усложнение.

Дело в том, что Clojure предлагает более эффективный способ добраться до
вложенных данных с известной структурой. Если мы точно знаем, что на вход
поступил вектор, второй элемент которого вектор, обратимся к get-in:

~~~clojure
(get-in [1 [2 3] 4] [1 1])
3
~~~

<!-- more -->

То же самое касается других типов данных. Неважно, какую какую комбинацию
образуют списки и словари: если структура известна заранее, до нужных данных
легко добраться с помощью `get-in` или стрелочного оператора. В данном случае
зипперы совершенно не нужны.

~~~clojure
(-> {:users [{:name "Ivan"}]}
    :users
    first
    :name)
"Ivan"
~~~

В чем же тогда преимущество зипперов? Свои сильные стороны они проявляют там,
где не может работать `get-in`. Речь о данных с неизвестной
структурой. Представьте, что на вход поступил произвольный вектор, и нужно найти
в нем строку. Она может быть на поверхности вектора, а может быть вложена на три
уровня. Другой пример — XML-документ. Нужный тег может располагаться где угодно,
и нужно как-то его найти. Другими словами, идеальный случай для зиппера —
нечеткая структура данных, о которой у нас только предположение.

Функции `zip/up`, `zip/down` и другие образуют универсальную `zip/next`. Эта
функция передвигает указатель так, что рано или поздно мы обойдем всю
структуру. При обходе исключены повторы: мы побываем в каждом месте только один
раз. Пример с вектором:

~~~clojure
(def vz (zip/vector-zip [1 [2 3] 4]))

(-> vz zip/node)
;; [1 [2 3] 4]

(-> vz zip/next zip/node)
;; 1

(-> vz zip/next zip/next zip/node)
;; [2 3]

(-> vz zip/next zip/next zip/next zip/node)
;; 2
~~~

Очевидно, мы не знаем, сколько раз вызывать `zip/next`, поэтому пойдем на
хитрость. Функция `iterate` принимает функцию `f` и значение `x`. Результатом
станет последовательность, где первый элемент `x`, а каждый следующий — `f(x)`
от предыдущего. Для зиппера мы получим исходную локацию, затем `zip/next` от
нее, затем `zip/next` от прошлого сдвига и так далее.

Ниже переменная `loc-seq` это цепочка локаций исходного зиппера. Чтобы получить
узлы, мы берем шесть первых элементов (число взяли случайно) и вызываем для
каждого `zip/node`.

~~~clojure
(def loc-seq (iterate zip/next vz))

(->> loc-seq
     (take 6)
     (map zip/node))

([1 [2 3] 4] 1 [2 3] 2 3 4)
~~~

`Iterate` порождает *ленивую* и *бесконечную* последовательность. Обе
характеристики важны. Ленивость означает, что очередной сдвиг (вызов `zip/next`)
не произойдет до тех пор, пока вы не дойдете до элемента в обработке
последовательности. Бесконечность означает, что `zip/next` вызывается
неограниченное число раз. Понадобится признак, по которому мы остановим вызов
`zip/next`, иначе обработка узлов никогда не закончится.

Пример ниже показывает, что в какой-то момент `zip/next` перестанет сдвигать
указатель. Добавьте или удалите `zip/next`, результат будет одинаков:

~~~clojure
(-> vz
    zip/next zip/next zip/next zip/next zip/next
    zip/next zip/next zip/next zip/next zip/next
    zip/next zip/next zip/next zip/next zip/next
    zip/node)
[1 [2 3] 4]
~~~

Функция `zip/next` устроена по принципу кольца. После последней локации она
перейдет на корневую и цикл завершится. При этом корневая локация получит
признак завершения, и дальнейший вызов `zip/next` ничего не даст. Проверить
признак можно функцией `zip/end?`:

~~~clojure
(def loc-end
  (-> [1 2 3]
      zip/vector-zip
      zip/next
      zip/next
      zip/next
      zip/next))

loc-end
[[1 2 3] :end]

(zip/end? loc-end)
~~~

Чтобы получить все локации, будем брать их до тех пор, пока локация не
конечна. Все вместе дает следующую функцию:

~~~clojure
(defn iter-zip [zipper]
  (->> zipper
       (iterate zip/next)
       (take-while (complement zip/end?))))
~~~

Эта функция вернет все локации в структуре данных. Напомним, что локация хранит
узел данных, который можно извлечь с помощью `zip/node`. Пример ниже показывает,
как превратить локации в данные:

~~~clojure
(->> [1 [2 3] 4]
     zip/vector-zip
     iter-zip
     (map zip/node))

([1 [2 3] 4]
 1
 [2 3]
 2
 3
 4)
~~~

Теперь когда мы получили цепочку локаций, напишем поиск. Предположим, нужно
проверить, есть ли в векторе кейворд `:error`. Напишем предикат для локации:

~~~clojure
(defn loc-error? [loc]
  (-> loc zip/node (= :error)))
~~~

Осталось проверить, если ли среди локаций та, чей узел равен `:error`. Для этого
подойдет `some` с нашим предикатом:

~~~clojure
(->> [1 [2 3 [:test [:foo :error]]] 4]
     zip/vector-zip
     iter-zip
     (some loc-error?))
true
~~~

Заметим, что из-за ленивости мы не сканируем все дерево. Если нужный узел
нашелся на середине, `iter-zip` обрывает итерацию, и дальнейшие вызовы
`zip/next` не срабатывают.

Будет полезно знать, что `zip/next` обходит дерево в глубину. При движении он
стремится вниз и вправо, а наверх поднимается лишь когда шаги в эти стороны
возвращают nil. Как мы увидим дальше, в некоторых случаях порядок обхода
важен. Попадаются задачи, где мы должны двигаться вширь. По умолчанию в
`clojure.zip` нет других вариантов обхода, но мы легко напишем
собственный. Позже мы рассмотрим задачу, где понадобится обход вширь.

Напишем свой зиппер для обхода вложенных словарей, например таких:

~~~clojure
(def map-data
  {:foo 1
   :bar 2
   :baz {:test "hello"
         :word {:nested true}}})
~~~

За основу возьмем стандартный `vector-zip` для векторов. Зипперы похожи, разница
лишь в типе коллекции. Подумаем, как задать функции-ответы на вопросы. Сам по
себе словарь — это ветка, чью потомки — элементы `MapEntry`. Последний тип это
пара ключа и значения. Если значение — словарь, получим из него цепочку
вложенных `MapEntry` и так далее. В Clojure нет встроенного предиката на
проверку `MapEntry`, поэтому напишем его:

~~~clojure
(def entry? (partial instance? clojure.lang.MapEntry))
~~~

Зиппер `map-zip` выглядит так:

~~~clojure
(defn map-zip [mapping]
  (zip/zipper
   (some-fn entry? map?)
   (fn [x]
     (cond
       (map? x) (seq x)
       (and (entry? x)
            (-> x val map?))
       (-> x val seq)))
   nil
   mapping))
~~~

При обходе дерева вы получите все пары ключей и значений. Если значение —
вложенный словарь, мы провалимся в него при обходе. Пример:

~~~clojure
(->> {:foo 42
      :bar {:baz 11
            :user/name "Ivan"}}
     map-zip
     iter-zip
     (map zip/node))

(...
 [:foo 42]
 [:bar {:baz 11, :user/name "Ivan"}]
 [:baz 11]
 [:user/name "Ivan"])
~~~

Многоточие выше заменяет исходный словарь. Чаще всего мы не интересуемся
корневым элементом, поэтому в конец стрелочного оператора можно добавить `rest`,
чтобы отбросить его.

С помощью этого зиппера можно проверить, есть ли в словаре ключ `:error` со
значением `:auth`. Сами по себе эти кейворды могут быть где угодно — и в ключах,
и в значениях на любом уровне. Однако нас интересует их комбинация. Для этого
напишем предикат:

~~~clojure
(defn loc-err-auth? [loc]
  (-> loc zip/node (= [:error :auth])))
~~~

Убедимся, что в первом словаре нет пары, даже не смотря на то, что значения
встречаются по отдельности:

~~~clojure
(->> {:response {:error :expired
                 :auth :failed}}
     map-zip
     iter-zip
     (some loc-err-auth?))
nil
~~~

И что их пара будет найдена:

~~~clojure
(->> {:response {:error :auth}}
     map-zip
     iter-zip
     (some loc-err-auth?))
true
~~~

На практике мы работаем с комбинацией векторов и словарей. Напишите свой зиппер,
который при обходе учитывает и словарь, и вектор.

(Продолжение следует)

{% include zipper-toc.md %}