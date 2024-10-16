Из [[Вывод типа шаблона | предыдущего раздела]]можно узнать почти все, что следует знать о выводе типа `auto`, поскольку за одним любопытным исключением вывод типа `auto` *представляет собой* вывод типа шаблона.
>[!question] Но как это может быть?
>Вывод типа шаблона работает с шаблонами, функциями и параметрами, а `auto` не имеет дела ни с одной из этих сущнотей.

Да, это так, но это не имеет значения. Существует прямая взаимосвязь между выводом типа шаблона и выводом типа `auto`. Существует буквальное алгоритмическое преобразование одного в другой.
```cpp
template<typename T>
void f(ParamType param);

f(expr); // вызов f с некоторым выражением
```
При вызове `f` компиляторы используют `expr` для вывода типов `T` и `ParamType`.

Когда переменная объявлена с использованием ключевого слова `auto`, оно играет роль `T`  в шаблоне, а спецификатор типа переменной действует как `ParamType`.
```cpp
auto x = 27;
```
Здесь спецификатором типа для `x` является `auto` само по себе. С другой стороны, в объявлении
```cpp
const auto cx = x;
```
спецификатором типа является `const auto`. А в объявлении
```cpp
const auto& rx = x;
```
спецификатором типа является `const auto&`. Для вывода типов для `x`, `cx`и `rx` в приведенных примерах компилятор действует так, как если бы для каждого объявления имелся шаблон, а также вызова этого шаблона с соответствующим инициализирующим выражением:
```cpp
template<typename T>      // концептуальный шаблон для
void func_for_x(T param); // вывода типа x

func_for_x(27);           // концептуальный вызов:
						  // выведенный тип param
						  // является типом x

template<typename T>             // концептуальный шаблон
void func_for_cx(const T param); // для вызова типа cx

func_for_cx(x);     // Концептуальный вызов:
                    // выведенный тип param является
                    // типом cx

template<typename T>              // концептуальный шаблон
void func_for_rx(const T& param); // для вызова типа rx

func_for_rx(x); // концептуальный вызов:
                // выведенный тип param является
                // типом rx
```
Вывод типов для `auto` представляет собой (с одним исключением, которое рассмотрим позже) то же самое, что и вывод типов для шаблонов.

В [[Вывод типа шаблона | предыдущем разделе]]вывод типов шаблонов был разделен на три случая, основанных на характеристиках `ParamType`, спецификаторе типа `param` в обобщенном шаблоне функции.

В объявлении переменной с использованием `auto` спецификатор типа занимает место `ParamType`, так что у нас опять имеются три случая.
- Случай 1. Спецификатор типа представляет собой ссылку или указатель, но не универсальную ссылку.
- Случай 2. Спецификатор типа представляет собой универсальную ссылку.
- Случай 3. Спецификатор типа не является ни ссылкой, ни указателем.

Мы уже встречались со случаями 1 и 3:
```cpp
auto         x = 27; // случай 3 (не указатель и не ссылка)
const auto  cx = x;  // случай 3 (не указатель и не ссылка)
const auto& rx = x;  // случай 1 (неуниверсальная ссылка)
```
Случай 2 работает, как и ожидалось:
```cpp
auto&& uref1 = x;   // x - int и lvalue, тип uref - int&
auto&& uref2 = cx;  // cx - const int и lvalue,
                    // тип uref2 - const int&
auto&& uref3 = 27;  // 27 - int и rvalue, так что тип
                    // uref3 - int&&
```
[[Вывод типа шаблона | Предыдущий раздел]] завершился обсуждением того, как имена массивов и функций превращаются в указатели для спецификаторов типа, не являющихся ссылками. То же самое происходит и при выводе типа `auto`:
```cpp
const char name[] = "R. N. Briggs"; // тип name - const char[13]
auto  arr1 = name; // тип arr1 - const char*
auto& arr2 = name; // тип arr2 - const char (&)[13]

void someFunc(int, double); // someFunc -- функция,
                            // ее тип void(int, double)

auto  func1 = someFunc; //  func1 - void (*)(int, double)
auto& func2 = someFunc; //  func2 - void (&)(int, double)
```
Как можно видеть, вывод типа `auto` работает подобно выводу типа шаблона. По сути это две стороны одной медали.

Они отличаются только в одном. Начнем с наблюдения, что если вы хотите объявить `int` с начальным значение 27, С++98 предоставляет вам две синтаксические возможности:
```cpp
int x1 = 27;
int x2(27);
```
C++11, поддерживая старые варианты инициализации, добавляет собственные:
```cpp
int x3 = {27};
int x4 {27};
```
Таким образом, у нас есть четыре разных синтаксиса, но результат один: пременная типа `int` со значением `27`.

Но объявление переменных с использованием ключевого слова `auto` вместо фисированных типов обладает определенными преимуществами, поэтому в приведенных выше объявлениях имеет смысл заменить `int` на `auto`. Простая замена текста приводит к следующему коду:
```cpp
auto x1 = 27;
auto x2(27);
auto x3 = {27};
auto x4 {27};
```
Все эти объявления компилируются, но их смысл оказывается не тем же, что и у объявлений, которые они заменяют.

Первые две инструкции в действительности объявляют переменную типа `int` со значением `27`.
Вторые две, однако, определяют переменную типа `std::initializer_list<int>`, содержащую единственный элемент со значением `27`.
```cpp
auto x1 = 27;   // тип int, начение - 27
auto x2(27);    // то же самое
auto x3 = {27}; // std::initializer_list<int>, yfxtybt 27
auto x4 {27};   // то же самое
```
Это объясняется специальным правилом вывода типа для `auto`. 
>[!note]
>Когда инициализатор для переменной, объявленной как `auto`, заключен в фигурные скобки, выведенный тип - `std::initializer_list`.
>

Если такой тип не может быть выведен (например, из-за того, что значения в фигурных скобках относятся к разным типам), код будет отвергнут:

```cpp
auto x5 = {1, 2, 3.0}; // Ошибка! Невозможно вывести T
                       // для std::initializer_list<T>
```
В этом случае вывод типа будет неудачным, но важно понимать, что на самом деле здесь имеют место два вывода типа.

Один из них вытекает из применения ключевого слова `auto`:  тип `x5` должен быть выведен. Поскольку инициализатор `x5` находится в фигурных скобках, тип `x5` должен быть выведен как `std::initializer_list`. Но `std::initializer_list` - это шаблон.

Конкретизация представляет собой создание `std::initializer_list<T>` с некоторым типом `T`, а это означает, что тип `T` также должен быть выведен.
Такой вывод относится ко второй разновидности вывода типов - выводу типа шаблона. В данном примере этот второй вывод неудачен, поскольку значения в фигурных скобках не относятся к одному и тому же типу.

==Рассмотрение инициализаторов в фигурных скобках является единственным отличием вывода типа `auto` от вывода типа шаблона.==
Когда объявленная с использованием ключевого слова `auto` переменная инициализируется с помощью инициализатора в фигурных скобках, выведенный тип представляет собой конкретизацию `std:initializer_list`.
Но если тот же инициализатор передается шаблону, вывод типа оказывается неудачным, и код отвергается:
```cpp
auto x = {11, 23, 9}; // Тип x - std::initializer_list<int>

template<typename T>  // Объявление шаблона с параметром
void f(T param);      // эквивалентно объялению x

f({11, 23, 9});  // Ошибка вывода типа T
```
Однако, если вы укажете в шаблоне, что `param` представляет собой `std::initializer_list<T>` для некоторого неизвестного `T`, вывод типа шаблона сможет определить, чем является `T`:
```cpp
template<typename T>
void f(std::initializer_list<T> initList);

f({11, 23, 9}); // Вывод int в качестве типа T, а тип initList - 
                // std::initializer_list<int>
```
>[!info]
>Таким образом, единственное реальное различие между выводом типа `auto` и выводом типа шаблона заключается в том, что `auto` *предполагает*, что инициализатор в фигурных скобках представляет собой `std::initializer_list`, в то время как вывод типа шаблона этого не делает.

Можно удивиться, почему вывод типа `auto` имеет специальное правило для инициализаторов в фигурных скобках, в то время как вывод типа шаблона такого правила не имеет. Но я и сам удивлен. Увы, я не в состоянии найти убедительное объяснение. Но "закон есть закон", и это означает, что нужно помнить, что если вы объявляете переменную с использованием ключевого слова `auto` и инициализируете ее с помощью инициализатора в фигурных скобках, то выводимым типом всегда будет `std::initializer_list`.
Особенно важно иметь это в виду, если вы приверженец философии унифицированной инициализации - заключения инициализирующих значений в фигурные скобки как само собой разумеющегося стиля.

С++14 и выше допускает применение `auto` для указания того, что возвращаемый тип функции должен быть выведен, а кроме того, лямбда-выражения могут использовать `auto` в объявлениях параметров. Однако такое применение `auto` использует вывод типа шаблона, а не вывод типа `auto`. Таким образом, функция с возвращаемым типом `auto`, которая возвращает инициализатор в фигурных скобках, компилироваться не будет.
```cpp
auto createInitList()
{
	return {1, 2, 3}; // Ошибка: невозможно вывести
	                  // тип для {1, 2, 3};
}
```
То же самое справедливо и тогда, когда `auto` используется в спецификации типа параметра в лямбда-выражении:
```cpp
std::vector<int> v;

auto resetV = 
	[&v](const auto& newValue) { v = newValue; }

resetV({1, 2, 3}); // Ошибка: невозможно вывести
                   // тип для {1, 2, 3}
```

>[!info] Следует запомнить
>- Вывод типа `auto` обычно такой же, как и вывод типа шаблона, но вывод типа `auto`, в отличие от вывода типа шаблона, предполагает, что инициализатор в фигурных скобках представляет `std::initializer_list`.
>- `auto` в возвращаемом типе функции или параметре лямбда-выражения влечет применение вывода типа шаблона, а не вывода типа `auto`.