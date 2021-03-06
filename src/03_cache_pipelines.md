# Кэш-память
- [Слайды с лекции](slides/lecture-3.pdf)
- [Запись лекции №1](https://www.youtube.com/watch?v=6vlNFxpSENs)
- [Запись лекции №2](https://www.youtube.com/watch?v=DddjrdCrCF8)
---
> Optimize for data first, then code. Memory access is probably going to be your biggest bottleneck

Одинаково ли по времени будет работать следующий код?

```python
for i = 0..n:				for i=0..n:
	for j=0..n:				for j=0..n:
		a[i][j] = 0 				a[j][i] = 0
```
У этих двух кодов большая разница из-за процессорного кэша. Первый цикл обращается к памяти последовательно, а второй "скачет" по ней. Поэтому обращения к памяти первого цикла попадают в кэш, а второго - нет. С ростом **N** видна разница между уровнями кэша на графике времени обработки одного элемента:

![Cache Hit Graph](./images/02.29_cache_hit_graph.png)

Небольшие пики - скачки из-за попадания в один бакет (заметно на степенях двойки), сильное изменение времени работы происходит, когда данные перестают попадать в кэш какого-то уровня.

**Кэш** реализован через хэш-таблицы (дискретного размера), где ключ - адрес в памяти.
Линии кэша примерно по *64 байта* разделенные на группы (buckets), размеры которых называются ассоциативностью кэша.

Подробнее про кэш  можно прочитать в [конспектах по ЭВМ](https://github.com/DespairedController/computer-architecture/blob/master/1_4/1_4.pdf) 

**Prefetching** - если много кэш-промахов, запрашиваем заранее подгрузить в кэш. Это работает на уровне кэш-подсистемы процессора, а не компилятора/ОС.

*Пример*: хранение хэш-таблицы с открытой адресацией. Два варианта: 

![Hash-Table](./images/02.29_hash_table.png)

- Если ожидаются частые попадания, то хранить полезнее данные рядом с ключом.

- Если ожидаются редкие попадания, то лучше хранить ключи и данные отдельно.

## TLB - translation lookaside buffer

Зачем нужен? Это кэш для ускорения трансляции виртуального адреса в физический. Так же, как и обычный, может быть нескольких уровней.

## Huge Pages

*Идея*: отображать страницу памяти в лежащую подряд физическую память. Тогда проще обходить каталог страниц. Например, если программе нужно прочитать 1 Мбайт непрерывных данных, которых нет в *TLB* кэше, то нужно сделать обращение к 256 страницам.

Но использовать тяжело, так как с ними тяжело делать *swap* (подкачка страниц). Например, в *Windows* требуются специальные права, чтобы выделять не-swappable память.

**Swap** - хотим выделить памяти больше, чем у нас есть физической (оперативной). Особенно было актуально раньше, когда оперативной памяти было мало. Некоторые страницы из оперативной памяти записывались на диск и помечались недоступными, но при необходимости можно было вытеснить другую страницу и выгрузить нужную с диска.


# Конвейер (Pipelining)

Ну тут опять всё [как было на ЭВМ](https://github.com/DespairedController/computer-architecture/blob/master/2_3-4/2_3-4.pdf).

Разбили выполнение команды на несколько стадий, теперь можем повысить частоту, так как каждая стадия стала проще. Выигрыш в том, что можем давать новые данные на каждом такте.

**Спекулятивное исполнение**: условные переходы дорогие, поэтому мы предсказываем переход, выполняем, а если не угадали, то откатываемя. В общем, ничего нового. Также это называется **branch prediction**.
Можем как выиграть, так и проиграть от этого. Например, [в некоторых программах на отсортированном массиве](https://stackoverflow.com/questions/11227809/why-is-processing-a-sorted-array-faster-than-processing-an-unsorted-array) предсказание может улучшить время работы в несколько раз.

Полезно писать программу так, чтобы уровень зависимостей команд был как можно меньше (это может также пытаться делать компилятор). Например:

```c++
int f(int a, int b, int c, int d)
{
 return a * b * c * d;
}
```

может скомпилиться в следующий код, чтоб уменьшить количество зависимых умножений:
` (a* b) * (c * d)`

```nasm
 imul		edi, esi
 imul		edx, ecx
 imul		edx, edi
 mov		eax, edx
 ret
```

Похожее происходит с циклами. Посмотрим на алгоритм Хаффмана:

 ```c++
void count_huffman_weights(char const* src, size_t size)
{
      uint32_t count[256] = {0};
      for (size_t i = 0; i != size; ++i)
          ++count[src[i]];
}
 ```

может быть разбит компилятором на параллельные исполнения. Так будут выглядеть зависимости:

![Dependencies](./images/02.29_dependencies.png)


Но у нас возникают проблемы из-за того, что мы пишем в одни переменные. Как пофиксить? Можем сделать 8 разных массивов-счетчиков. Такая реализация используется в библиотеке **Zstandart**

```c++
void count_huffman_weights_improved(char const* src, size_t size)
{
 uint32_t count[8][256] = {};
 size = size / 8 * 8;
 for (size_t i = 0; i < size;)
     {
           ++count[0][src[i++]]; ++count[1][src[i++]]; ++count[2][src[i++]];
           ++count[3][src[i++]]; ++count[4][src[i++]]; ++count[5][src[i++]];
           ++count[6][src[i++]]; ++count[7][src[i++]];
     }
}
```

-----------
​	Полезные книжки по теме:

- J. Shen, M. Lipasti — Modern Processor Design: Fundamentals of Superscalar Processors
- J. Hennessy, D. Patterson — Computer Architecture: A Quantitative Approach

