### Интеграция оптимизаций трёхадресного кода между собой
#### Постановка задачи
Необходимо скомбинировать созданные ранее оптимизации трёхадресного кода так, чтобы они могли выполняться все вместе, друг за другом.
#### Команда
Д. Володин, Н. Моздоров
#### Зависимые и предшествующие задачи
Предшествующие: 
- Def-Use информация: накопление информации и удаление мертвого кода на ее основе
- Устранение переходов к переходам
- Очистка кода от пустых операторов
- Устранение переходов через переходы
- Учет алгебраических тождеств
- Живые и мертвые перем и удаление мертвого кода (замена на пустой оператор)
- Оптимизация общих подвыражений
- Протяжка констант
- Протяжка копий
- Разбиение трёхадресного кода на базовые блоки

#### Теоретическая часть
Необходимо организовать выполнение оптимизаций трёхадресного кода до тех пор, пока каждая из созданных оптимизаций перестанет изменять текущий список инструкций.

#### Практическая часть
Для данной задачи был создан статический класс `ThreeAddressCodeOptimizer`, содержащий два публичных метода: `Optimize` и `OptimizeAll`. Первый метод на вход получает список инструкций, а также два списка оптимизаций: те, которые работают в пределах одного базового блока, и те, которые работают для всего кода программы. Параметрам - спискам оптимизаций по умолчанию присвоено значение `null`, что позволяет при вызове указывать только один из списков. Второй метод на вход получает только список инструкций и использует оптимизации, хранящиеся в двух приватных списках внутри класса, содержащих все созданные оптимизации трёхадресного кода.

Оптимизация выполняется следующим образом: сначала список инструкций делится на базовые блоки, затем для каждого блока отдельно выполняются все оптимизации в пределах одного блока, затем инструкции в блоках объединяются и выполняются все глобальные оптимизации. Общая оптимизация в пределах одного блока и общая оптимизация всего кода выполняются похожим образом и представляют собой циклы, пока все соответствующие оптимизации не перестанут изменять список инструкций, и в этих циклах по очереди выполняется каждая из соответствующих оптимизаций. Если какая-то из оптимизаций изменила список инструкций, то выполнение всех оптимизаций происходит заново. Ниже приведён код для общей оптимизации в пределах одного блока.
```csharp
private static BasicBlock OptimizeBlock(BasicBlock block, List<Optimization> opts)
{
    var result = block.GetInstructions();
    var currentOpt = 0;
    while (currentOpt < opts.Count)
    {
        var (wasChanged, instructions) = opts[currentOpt++](result);
        if (wasChanged)
        {
            currentOpt = 0;
            result = instructions;
        }
    }
    return new BasicBlock(result);
}
```

#### Место в общем проекте (Интеграция)
Данная оптимизация объединяет созданные ранее оптимизации трёхадресного кода, и в дальнейшем на основе результата выполнения всех оптимизаций выполняется построение графа потока управления.
#### Тесты
Класс `ThreeAddressCodeOptimizer` используется во всех тестах для проверки оптимизаций трёхадресного кода (в том числе тех оптимизаций, которые дополняют действие друг друга). Схема тестирования выглядит следующим образом: сначала по заданному тексту программы генерируется трёхадресный код, затем задаются списки оптимизаций для проверки, после этого вызывается метод `Optimize` класса `ThreeAddressCodeOptimizer` и сравнивается полученный набор инструкций с ожидаемым набором. Ниже приведён один из тестов. 
```csharp
[Test]
public void VarAssignSimple()
{
    var TAC = GenTAC(@"
var a, b, x;
x = a;
x = b;
");
    var optimizations = new List<Optimization> { ThreeAddressCodeDefUse.DeleteDeadCode };

    var expected = new List<string>()
    {
        "noop",
        "x = b"
    };
    var actual = ThreeAddressCodeOptimizer.Optimize(TAC, optimizations)
        .Select(instruction => instruction.ToString());

    CollectionAssert.AreEqual(expected, actual);
}
```