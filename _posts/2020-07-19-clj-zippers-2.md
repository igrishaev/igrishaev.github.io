---
layout: post
title:  "Зипперы в Clojure (часть 2). Автонавигация"
permalink: /clj-zippers-2/
tags: clojure zippers
---

{% include zipper-toc.md %}

Мы разобрались с тем, как перемещаться по коллекции. Однако у читателя
возникнет вопрос: откуда приходит путь? Как мы узнаем заранее, в каком
направлении двигаться?

Выскажем главный тезис этой главы. **Ручная навигация по данным лишена всякого
смысла.** Если путь известен заранее, вам не нужен зиппер — это лишнее
усложнение.

Дело в том, что Clojure предлагает более простой способ работы с данными,
структура которых известна. Например, если мы точно знаем, что на вход поступил
вектор, второй элемент которого вектор, обратимся к `get-in`:

~~~clojure
(def data [1 [2 3] 4])

(get-in data [1 1])
;; 3
~~~

<!-- more -->

То же самое касается других типов данных. Неважно, какую какую комбинацию
образуют списки и словари: если структура известна заранее, до нужных данных
легко добраться с помощью `get-in` или стрелочного оператора. В данном случае
зипперы только усложнят код.

~~~clojure
(def data {:users [{:name "Ivan"}]})

(-> data :users first :name)
;; "Ivan"
~~~

В чем же тогда преимущество зипперов? Свои сильные стороны они проявляют там,
где не может работать `get-in`. Речь о данных с *неизвестной*
структурой. Представьте, что на вход поступил произвольный вектор, и нужно найти
в нём строку. Она может быть как на поверхности вектора, так и вложена на три
уровня. Другой пример — XML-документ. Нужный тег может располагаться где угодно,
и нужно как-то его найти. Другими словами, идеальный случай для зиппера —
нечёткая структура данных, о которой у нас только предположения.

Функции `zip/up`, `zip/down` и другие образуют универсальную `zip/next`. Эта
функция передвигает указатель так, что рано или поздно мы обойдем всю
структуру. При обходе исключены повторы: мы побываем в каждом месте только один
раз. Пример с вектором:

~~~clojure
(def vzip (zip/vector-zip [1 [2 3] 4]))

(-> vzip zip/node)
;; [1 [2 3] 4]

(-> vzip zip/next zip/node)
;; 1

(-> vzip zip/next zip/next zip/node)
;; [2 3]

(-> vzip zip/next zip/next zip/next zip/node)
;; 2
~~~

Очевидно, мы не знаем, сколько раз вызывать `zip/next`, поэтому пойдём на
хитрость. Функция `iterate` принимает функцию `f` и значение `x`. Результатом
станет последовательность, где первый элемент `x`, а каждый следующий — `f(x)`
от предыдущего. Для зиппера мы получим исходную локацию, затем `zip/next` от
неё, затем `zip/next` от прошлого сдвига и так далее.

Ниже переменная `loc-seq` -- это цепочка локаций исходного зиппера. Чтобы получить
узлы, мы берём шесть первых элементов (число взяли случайно) и вызываем для
каждого `zip/node`.

~~~clojure
(def loc-seq (iterate zip/next vzip))

(->> loc-seq
     (take 6)
     (map zip/node))

;; ([1 [2 3] 4]
;;   1
;;   [2 3]
;;   2
;;   3
;;   4)
~~~

`Iterate` порождает *ленивую* и *бесконечную* последовательность. Обе
характеристики важны. Ленивость означает, что очередной сдвиг (вызов `zip/next`)
не произойдёт до тех пор, пока вы не дойдёте до элемента в
цепочке. Бесконечность означает, что `zip/next` вызывается неограниченное число
раз. Понадобится признак, по которому мы остановим вызов `zip/next`, иначе поток
локаций никогда не закончится.

Кроме того, если исследовать каждую локацию, станет ясно, что в какой-то момент
`zip/next` уже не сдвигает указатель. Возьмём наугад сотый и тысячный элементы
итерации. Их узел будет исходным вектором:

~~~clojure
(-> loc-seq (nth 100) zip/node)
;; [1 [2 3] 4]

(-> loc-seq (nth 1000) zip/node)
;; [1 [2 3] 4]
~~~

Причина в том, как устроен обход зиппера. Функция `zip/next` работает по
принципу кольца. Когда она достигает исходной локации, цикл завершается. При
этом локация получит признак завершения, и дальнейший вызов `zip/next` вернёт её
же. Проверить признак можно функцией `zip/end?`:

~~~clojure
(def loc-end
  (-> [1 2 3]
      zip/vector-zip
      zip/next
      zip/next
      zip/next
      zip/next))

loc-end
;; [[1 2 3] :end]

(zip/end? loc-end)
;; true
~~~

Чтобы получить конечную цепь локаций, будет сдвигать указатель до тех пор, пока
локация не последняя. Всё вместе даёт следующую функцию:

~~~clojure
(defn iter-zip [zipper]
  (->> zipper
       (iterate zip/next)
       (take-while (complement zip/end?))))
~~~

Эта функция вернёт все локации в структуре данных. Напомним, что локация хранит
узел (элемент данных), который можно извлечь с помощью `zip/node`. Пример ниже
показывает, как превратить локации в данные:

~~~clojure
(->> [1 [2 3] 4]
     zip/vector-zip
     iter-zip
     (map zip/node))

;; ([1 [2 3] 4]
;;  1
;;  [2 3]
;;  2
;;  3
;;  4)
~~~

Теперь когда мы получили цепочку локаций, напишем поиск. Предположим, нужно
проверить, есть ли в векторе кейворд `:error`. Сначала напишем предикат для
локации -- равен ли её узел этому значению.

~~~clojure
(defn loc-error? [loc]
  (-> loc zip/node (= :error)))
~~~

Осталось проверить, если ли среди локаций та, что подходит нашему предикату.
Для этого вызовем `some`:

~~~clojure
(def data [1 [2 3 [:test [:foo :error]]] 4])

(some loc-error?
      (-> data zip/vector-zip iter-zip))

;; true
~~~

Заметим, что из-за ленивости мы не сканируем всё дерево. Если нужный узел
нашелся на середине, `iter-zip` обрывает итерацию, и дальнейшие вызовы
`zip/next` не сработают.

Будет полезно знать, что `zip/next` обходит дерево в глубину. При движении он
стремится вниз и вправо, а наверх поднимается лишь когда шаги в эти стороны
возвращают nil. Как мы увидим дальше, в некоторых случаях порядок обхода
важен. Попадаются задачи, где мы должны двигаться вширь. По умолчанию в
`clojure.zip` нет других вариантов обхода, но мы легко напишем
собственный. Позже мы рассмотрим задачу, где понадобится обход вширь.

Встроенный зиппер `vector-zip` служит для вложенных векторов. Но гораздо чаще
встречаются вложенные словари. Напишем свой зиппер для обхода подобных данных:

~~~clojure
(def map-data
  {:foo 1
   :bar 2
   :baz {:test "hello"
         :word {:nested true}}})
~~~

За основу возьмём знакомый нам `vector-zip`. Зипперы похожи, разница лишь в типе
коллекции. Подумаем, как задать функции-ответы на вопросы. Сам по себе словарь —
это ветка, чью потомки — элементы `MapEntry`. Этот тип выражает пару ключа и
значения. Если значение — словарь, получим из него цепочку вложенных `MapEntry`
и так далее.

Для разминки напишем предикат на проверку типа `MapEntry`:

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

Поясним основные моменты. Композиция `(some-fn ...)` вернёт истину, если один из
параметров-предикатов сработает положительно. Иными словами, на роль ветки мы
рассматриваем только словарь или его узел (пару ключ-значение).

Во второй функции, которая ищет потомков, приходится делать перебор. Для словаря
получим его потомков функцией `seq` -- она вернёт цепочку элементов
`MapEntry`. Если мы уже находимся в `MapEntry`, то проверим, является ли
значение вложенным словарём. Если да, получим его потомков той же функцией `seq`.


При обходе дерева вы получите все пары ключей и значений. Если значение —
вложенный словарь, мы провалимся в него при обходе. Пример:

~~~clojure
(->> {:foo 42
      :bar {:baz 11
            :user/name "Ivan"}}
     map-zip
     iter-zip
     rest
     (map zip/node))

;; ([:foo 42]
;;  [:bar {:baz 11, :user/name "Ivan"}]
;;  [:baz 11]
;;  [:user/name "Ivan"])
~~~

Обратите внимание на функцию `rest` после `iter-zip`. Мы отбросили первую
локацию, в которой находятся исходные данные. Поскольку они и так известны, нет
смысла выводить их на печать.

С помощью нашего `map-zip` можно проверить, есть ли в словаре ключ `:error` со
значением `:auth`. По отдельности эти кейворды могут быть где угодно — и в
ключах, и в значениях на любом уровне. Однако нас интересует их комбинация. Для
этого напишем предикат:

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

;; nil
~~~

Но даже если пара вложена глубоко, мы найдём её:

~~~clojure
(def data
  {:response {:info {:message "Auth error"
                     :error :auth
                     :code 1005}}})

(->> data
     map-zip
     iter-zip
     (some loc-err-auth?))

;; true
~~~

Несколько заданий для самостоятельной работы.

1) Зиппер `map-zip` не учитывает случай, когда ключ словаря -- другой словарь,
например:

{% raw %}
~~~clojure
{{:alg "MD5" :salt "***"} "deprecated"
 {:alg "SHA2" :salt "****"} "deprecated"
 {:alg "HMAC-SHA256" :key "xxx"} "ok"}
~~~
{% endraw %}

Такие коллекции хоть и редко, но встречаются в практике. Доработайте `map-zip`,
чтобы он проверял не только значение `MapEntry`, но и ключ.

2) На практике мы работаем с комбинацией векторов и словарей. Напишите
универсальный зиппер, который учитывает при обходе и словарь, и вектор.

(Продолжение следует)

{% include zipper-toc.md %}
