# Массивы и слайсы

## Массив

По определению тип массива состоит из длины и типа его элементов. Например, тип `[4]int` представляет массив из четырёх целых чисел. Размер массива неизменяем; его *длина - это часть его типа* (`[4]int` и `[5]int` различные, несовместимые типы). Массивы могут быть проиндексированы, поэтому с помощью выражения `s[n]` мы получаем доступ к n-ному элементу, начиная с нуля. Массивы не нужно инициализировать явно; нулевой массив - это готовый к использованию массив, элементы которого являются нулями:

```go
var a [4]int
a[0] = 1
// a[2] == 0, нулевое значение типа int
```



Представление [4]int в памяти - это просто четыре целых значения, расположенных последовательно:

<img src="..\..\..\src\media\go\slice-array.png" max-width="100%"/>

Массивы в Go и есть значения. Переменная с именем массива обозначает весь массив; это не указатель на первый элемент (как это было бы в случае С). Это значит, что когда вы присваиваете значение или проходитесь по массиву, вы будете делать копию его содержимого (для избежания копирования, вы могли бы передавать указатель на массив, но тогда это будет указатель на него, а не сам массив). 

Литерал массива может быть задан так + можете указать компилятору посчитать количество значений:

```go
b := [2]string{"Penn", "Teller"}
с := [...]string{"Penn", "Teller"}
```

## Слайсы

Слайс - это дескриптор сегмента массива. Он состоит из указателя на массив, длины сегмента и его вместимости (максимальной длины сегмента).

```go
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```

Наша переменная `s`, созданная ранее с помощью `make([]byte, 5)`, имеет такую структуру:



<img src="..\..\..\src\media\go\slice-1.png" max-width="100%"/>

Длина - это число элементов, на которое ссылается слайс. Вместимость - это число элементов лежащего в основе массива (начиная с элемента, на который ссылается указатель слайса). Разница между длиной и вместимостью станет чётче по ходу знакомства с остальными примерами.

По мере изменения промежутков слайса, можно наблюдать изменения в структуре данных слайса и их отношениях с лежащим в основе массивом:

```go
s = s[2:4]
```

<img src="..\..\..\src\media\go\slice-2.png" max-width="100%"/>

Слайсниг не производит копирование данных слайса. Создаётся новое значение слайса, указывающее на исходный массив. Это делает операцию слайсинга такой же эффективной, как и манипуляции с индексами массива. Таким образом, изменение элементов (не самого слайса) нового слайса изменяет элементы исходного:

```go
d := []byte{'r', 'o', 'a', 'd'}
e := d[2:] 
// e == []byte{'a', 'd'}
e[1] = 'm'
// e == []byte{'a', 'm'}
// d == []byte{'r', 'o', 'a', 'm'}
```

Ранее мы слайсили `s` до длины, меньшей, чем вместимость. Мы можем увеличить `s` до её вместимости, сделав слайсинг снова. Слайс нельзя сделать большим, чем его вместимость. Если вы попытаетесь, это вызовет панику времени выполнения, как и когда происходит обращение к индексу вне границ слайса или массива.

### Увеличение слайсов (функции copy и append)

Для увеличения вместимости слайса необходимо создать новый, более крупный слайс и скопировать элементы исходного слайса в него. Эта техника показывает, как реализуются динамические массивы в других языках. Следующий пример удваивает вместимость `s`, создавая новый слайс `t`, копируя содержимое `s` в `t`, а затем присваивая `s` значение слайса `t`:

```go
t := make([]byte, len(s), (cap(s)+1)*2) // +1 в случае cap(s) == 0
for i := range s {
        t[i] = s[i]
}

s = t
```

Повторяющаяся часть этой часто используемой операции реализована с помощью простой встроенной функции `copy`. Как подсказывает её имя, эта функция копирует данные из слайса-источника в слайс-приёмник. Возвращается количество скопированных элементов.

```go
func copy(dst, src []T) int
```

Функция `copy` поддерживает копирование между слайсами разной длины (она скопирует только до меньшего числа элементов). К тому же, `copy` может справиться со слайсами, относящимися к одному массиву в основе этих слайсов, работая правильно с перекрытием слайсов.

Используя `copy`, можно упростить кусочек кода выше:

```go
t := make([]byte, len(s), (cap(s)+1)*2)
copy(t, s)
s = t
```

Часто необходимо добавить данные в конец слайса. Эта функция добавляет элементы в байтовый слайс, увеличивая сам слайс по необходимости, и возвращает обновлённый слайс:

```go
func AppendByte(slice []byte, data ...byte) []byte {
    m := len(slice)
    n := m + len(data)
    if n > cap(slice) { // если нужно, перераспределить память
        // выделяем в два раза больше нужного, для увеличения в будущем
        newSlice := make([]byte, (n+1)*2)
        copy(newSlice, slice)
        slice = newSlice
    }
    slice = slice[0:n]
    copy(slice[m:n], data)
    return slice
}
```

Можно было бы использовать AppendByte таким образом:

```go
p := []byte{2, 3, 5}
p = AppendByte(p, 7, 11, 13)
// p == []byte{2, 3, 5, 7, 11, 13}
```

Такие функции, как AppendByte, полезны, потому что они предоставляют полный контроль над способом увеличения слайсов. В зависимости от характеристики программы может понадобиться создание более маленького или большого слайса, или загрузить слайс элементами до предельного размера памяти.

Хотя большинству программ не нужен абсолютный контроль, поэтому Go предоставляет встроенную функцию `append`, которая хорошо подходит в большинстве случаев. Она имеет такую сигнатуру:

```go
func append(s []T, x ...T) []T
```

Эта функция добавляет элементы в конец слайса `s` и увеличивает вместимость, если нужно.

```go
a := make([]int, 1)
// a == []int{0}
a = append(a, 1, 2, 3)
// a == []int{0, 1, 2, 3}
```

Чтобы добавить один слайс в другой, используйте `…` в качестве второго аргумента, чтобы он стал списком аргументов.

```go
a := []string{"John", "Paul"}
b := []string{"George", "Ringo", "Pete"}
a = append(a, b...) // то же самое, что и "append(a, b[0], b[1], b[2])"
// a == []string{"John", "Paul", "George", "Ringo", "Pete"}
```

Так как нулевой слайс работает как слайс нулевой длины, вы можете объявить переменную со слайсом и затем циклично добавлять в неё элементы:

```go
// Filter возвращает новый слайс,
// из элементов s, удовлетворяющих условиям функции f()
func Filter(s []int, fn func(int) bool) []int {
    var p []int // == nil
    for _, v := range s {
        if fn(v) {
            p = append(p, v)
        }
    }
    return p
}
```

### Возможная ловушка

Как говорилось ранее, переслайсинг (re-slicing) среза не создаёт копию массива в основании. Массив полностью будет существовать в памяти, пока на него не перестанут ссылаться. Иногда это вызывает хранение всех данных в памяти, когда нужна только их небольшая часть.

Например, функция `FindDigits` загружает файл в память и ищет в нём первую группу последовательных цифр, возвращая их в новом слайсе.

```go
var digitRegexp = regexp.MustCompile("[0-9]+")

func FindDigits(filename string) []byte {
    b, _ := ioutil.ReadFile(filename)
    return digitRegexp.Find(b)
}
```

Этот код работает, как и говорилось, однако возвращаемый срез `[]byte` указывает на массив, содержащий файл целиком. Так как слайс ссылается на исходный массив, пока слайс есть в памяти, сборщик мусора не сможет очистить массив; несколько важных байтов файла держат всё содержимое в памяти.

Чтобы исправить это, можно скопировать интересующие нас данные в новый слайс до того, как вернуть значение.

```go
func CopyDigits(filename string) []byte {
    b, _ := ioutil.ReadFile(filename)
    b = digitRegexp.Find(b)
    c := make([]byte, len(b))
    copy(c, b)
    return c
}
```

###  