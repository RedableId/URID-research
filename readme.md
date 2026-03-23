# TLTR

Задача данного проекта -- найти наиболее эффективную упаковку строки-идентификатора в 128-битный векторы (64, 192, 256 так же могут быть использованы, но 128 -- наиболее практичный размер сопоставимый с расходами на хранение пустой классической строки).

Идентификатор -- это уникальное название сущности. Эти названия часто удобны при разработке и дебаге, но во время исполнения их строковое значение редко бывает полезно (конечно бывает, например для загрузки ресурса по имени или для создания калюча локализации).

Многие хотели бы не хранить эти строки в куче, не пересоздавать одинаковые строки и избежать других накладных расходов работы со строковымми данными.

Этот проект поможет справиться вам с некоторыми перечисленными проблемами, если вы не против некоторых ограничений.

# Детали

## Терминология

* __Буква__ -- символ из диапазона 'a' .. 'z';
* __Цифра__ -- символ из диапазона '0' .. '9';
* __Токен__ -- последовательность символов только букв или только цифр;
* __Слово__ -- токен состоящий из букв;
* __Число__ -- токен состоящий из цифр;
* __Разделитель__ -- место в идентификаторе, означающее конец предыдущего токена и начало следующего (может быть отдельным символом или быть определенным по контексту).
* __Идентификатор__ -- последовательность токенов;
* __Путь__ -- последовательность идентификаторов.
* __Вектор__ -- N-битное значение, содержащее идентификатор, где N может быть 64, 128, 192 или 256.

## Язык

Используются символы английского алфавита и арабские цифры. Все другие символы считаются разделителями. Так же разделителем считается граница между буквами и цифрами без явного символа-разделителя (например `unit31` -- это два токена `unit` и `31`). В случае использования формата camelCase или PascalCase, граница определяется переходом от строчной к заглавной букве (например `unitBoss` -- это два токена `unit` и `Boss`).

## Эффективность

Под эффективностью понимается наиболее практичная реализация, позволяющая умещать в векторе наибольшее число символов, при этом позволяющая изменять вектор с наименьшими вычислительными затратами.

## Требования

Длина идентификатора упакованного в фиксированный вектор в любом случае ограничена. Это ограничение может быть фиксировано или меняться в зависимости от количества токенов или чего-то еще. Наиболее практичным был бы подход при котором ограничение наиболее очевидно пользователю.

Числа в идентификаторах -- спорный момент. Рассматриваются варианты:
* `Free` -- Числа разрешены в любом месте идентификатора;
* `EndMulti` -- Числа могут встречаться только в конце идентификатора;
* `EndSingle` -- Только одно число может встречаться в конце идентификатора;

Было бы очень удобно, если пустрая строка кодировалась в нулевой вектор.

## Операции

Операции, перечисленные в порядке приоритета минимизации их вычислительных затрат:

* `Encode` -- Упаковка строки в вектор
* `Decode` -- Распаковка строки из вектора
* `Get` -- Взятие токена по индексу
* `Replace` -- Замена токена по индексу
* `Insert` -- Вставка токена по индексу
* `Remove` -- Удаление токена по индексу

## Results

### Terms
* $D_b$ -- Best bits per symbol;
* $D_w$ -- Worst bits per symbol;
* $L_{vec}$ -- Vector length in bits;
* $L_{id}$ -- Identifier length in symbols;
* $L_{token}$ -- Token length in symbols;
* $C_{token}$ -- Tokens count;

### Facts
* $L_{token} \leq L_{id}$ -- token can't be longer than whole identifier;
* $C_{token} \leq L_{id}$ -- tokens count can't be more than identifier length;
* $\overline{L_{token}} * C_{token} \leq L_{id}$ -- average token length and tokens count cannot be large both at the same time;

### Complexity
We interested in complexity of the following operations:
- Identifier operations:
    - Encode -- complexity of encoding of identifier of length $L_{id}$ into vector of length $L_{vec}$;
    - Decode -- complexity of decoding of vector of length $L_{vec}$ into identifier of length $L_{id}$;
    - GetLength -- complexity of getting identifier length in symbols;
- Token operations:
    - AppendSymbol -- complexity of adding a symbol to the end of identifier;
    - AppendToken -- complexity of adding a token to the end of identifier;
- Symbol operations:
    - GetSymbol -- complexity of getting symbol by index;
    - GetToken -- complexity of getting token by index;
    - ReplaceSymbol -- complexity of replacing symbol by index;
- Complexity of Insert/Remove of Symbol/Token by index is expected to be the same as for string buffer.

Most complexity is $O(L_{id})$ -- that means in worst case we need decode (and re-encode sometimes) whole identifier.
By following table we highlight only interesting cases of complexity.

### Comparison table

| Approach             | $D_b$ | $D_w$ | Complexity | Comment |
|:---------------------|:-----:|:-----:|:----------:|:-------:|
| Enumeration-EndMulti | 4.536 | 4.536 | Everything slow | Requires full reencoding on any change |
| radix-token          | 4.832 | 9.142 | Fast token operations | Capacity decreases with more tokens |
| Base32               | 5.000 | 5.000 | Fast symbol operations | Loses token's bounds |
| Base3-13-29          | 5.120 | 9.846 | Everything slow | Capacity decreases with more tokens |
| Base37               | 5.333 | 5.333 | Fast symbol operations | Magic-mul instead of bits-shift for division |
| Base40               | 5.333 | 5.333 | Fast symbol operations | Same values can be encoded in different ways without strict rules |
| Base64               | 6.000 | 6.000 | Fast symbol operations | Perfectly fits to 192-bits vector |