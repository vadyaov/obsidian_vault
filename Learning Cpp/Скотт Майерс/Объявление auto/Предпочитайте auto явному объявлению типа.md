Давайте порадуемся объявлению локальной переменной, инициализированной разыменованием итератора:
```cpp
template<typename It>
void dwim(It b, It e)
{
	while (b != e) {
		typename std::iterator_traits<It>::value_type
		    currValue = *b;
	}
}
```
Жуть. `typename std::iterator_traits<It>::value_type` - просто чтобы записать тип значения, на которое указывает итератор?
С таким же успехом можно попробовать объявить локальную переменную,  тип которой такой же, как у лямбда-выражения. Но его тип известен только компилятору!
Никакого удовольствия от программирования на С++!

В С++11 все эти проблемы решены с помощью ключевого слова `auto`. Тип переменных, объявленных как `auto`, выводится из инициализатора, так что они обязаны быть инициализированными. Это значит - прощай проблема неинициализированных переменных:
```cpp
int x1;       // Потенциально неинициализированная переменная
auto x2;      // Ошибка! Требуется инициализатор
auto x3 = 0;  // Все отлично, переменная x корректно определена
```
Нет проблем с объявлением локальной переменной, значением которой является разыменование итератора:
```cpp
template<typename It>
void dwim(It b, It e)
{
	while (b != e) {
		auto currValue = *b;
	}
}
```
А поскольку `auto` использует вывод типов (см. раздел [[Вывод типа auto]]), он может представлять типы, известные только компиляторам:
```cpp
auto derefUPLess =                    // Функция сравнения
[](const std::unique_ptr<Widget>& p1, // для значений, на
   const std::unique_ptr<Widget>& p2) // которые указывают
   { return *p1 < *p2; };             // std::unique_ptr
```
Несмотря на всю крутость вы, вероятно, думаете, что можно обойтись и без `auto` для объявления переменной, которая хранит лямбда-выражение, поскольку мы можем использовать объект `std::function`. Это так, можем, но, возможно, это не то, что вы на самом деле подразумеваете. Давайте разбираться.