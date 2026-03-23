# URID - Universal Readable Identifier

Задача данного проекта -- найти наиболее эффективную упаковку строки-идентификатора в 128-битный векторы (64, 192, 256 так же могут быть использованы, но 128 -- наиболее практичный размер сопоставимый с расходами на хранение пустой классической строки).

Идентификатор -- это уникальное название сущности. Эти названия часто удобны при разработке и дебаге, но во время исполнения их строковое значение редко бывает полезно (но, конечно бывает, например для загрузки ресурса по имени или для создания калюча локализации).

Многие хотели бы не хранить эти строки в куче, не пересоздавать одинаковые строки и избежать других накладных расходов работы со строковымми данными.

Этот проект поможет справиться вам с некоторыми перечисленными проблемами, если вы не против некоторых ограничений.

# Details

## Terminology

* __Letter__ -- a character in the range 'a' .. 'z';
* __Digit__ -- a character in the range '0' .. '9';
* __Symbol__ -- Letter or Digit;
* __Token__ -- a sequence of symbols that are either all letters or all digits;
* __Word__ -- a token consisting of letters;
* __Number__ -- a token consisting of digits;
* __Separator__ -- a position in the identifier indicating the end of the previous token and the beginning of the next (can be a separate character or determined by context).
* __Identifier__ -- a sequence of tokens;
* __Path__ -- a sequence of identifiers.
* __Vector__ -- an N-bit value describing an identifier, where N can be 64, 128, 192, or 256.

## Language

Identifiers use characters from the English alphabet and Arabic numerals. All other characters are considered separators.

A separator is also considered to be the boundary between letters and digits without an explicit separator character (for example, `unit31` is two tokens `unit` and `31`). In the case of camelCase or PascalCase format, the boundary is determined by the transition from lowercase to uppercase letter (for example, `unitBoss` is two tokens `unit` and `Boss`).

## Identifier format

Encoded vector value is format-agnostic. The user decides how to interpret the vector value as an identifier. We support case-based and separator-based formats. For example, for the same vector value we can have:
* `greenBuyButton` -- camelCase format;
* `GreenBuyButton` -- PascalCase format;
* `green_buy_button` -- snake_case format;
* `green-buy-button` -- kebab-case format;
* etc...

## Efficiency

Efficiency is defined as the most practical implementation that allows to fit the largest number of symbols in a vector.

Also operations of full decoding and encoding are in priority. Fast random access to tokens and symbols is a nice-to-have, but not a must-have.

## Requirements

### Identifier length
The length of an identifier packed into a fixed vector is limited in any case. This limitation can be fixed or vary depending on the number of tokens or something else. The most practical approach would be one where the limitation is most obvious to the user.

### Identifier expressiveness
There is no right opinion about usage of numbers and separators in identifiers. Considering modes:
* `Default` -- Any combination of letters, digits and separators is allowed (ex. `_button_x3_dark__`);
* `ES` (explicit-separator) -- Separator is allowed only when there is no way to determine token boundaries by context (ex. `unit31`, but `unit_dog`, or `x64_86`);
* `EMN` (ends-multiple-numbers) -- Multiple numbers are allowed, but only at the end of the identifier;
* `ESN` (ends-single-number) -- Only a single number is allowed at the end of the identifier;

### Vector values
Zero vector must represent an empty identifier.

Any value of vector must be decodable into a valid identifier.

For any vector value $V$ is truth that $encode(decode(V)) = V$.

## Operations

Operations, listed in order of priority for minimizing their computational cost:

* `Decode` -- Unpacking a string from a vector
* `Encode` -- Packing a string into a vector
* `Get` -- Getting a token by index
* `Replace` -- Replacing a token by index
* `Insert` -- Inserting a token by index
* `Remove` -- Removing a token by index

## Results

### Terms
* $D_b$ -- Best bits per symbol;
* $D_a$ -- Average bits per symbol;
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

| Approach             | $D_b$ | $D_a$ | $D_w$ | Complexity | Comment |
|:---------------------|:-----:|:-----:|:-----:|:----------:|:-------:|
| tANS (Tabled ANS)    | 3.434 | 4.492 | 9.542 | Fast encoding/decoding, slow random access | Ground-truth of coding entropy |
| Enumeration-EMN      | 4.536 | 4.536 | 4.536 | Everything slow | Requires full re-encoding on any change |
| radix-token          | 4.832 | ?.??? | 9.142 | Fast token operations | Capacity decreases with more tokens |
| Base32               | 5.000 | 5.000 | 5.000 | Fast symbol operations | Loses token's bounds |
| Base3-13-29          | 5.120 | ?.??? | 9.846 | Everything slow | Capacity decreases with more tokens |
| Base37               | 5.333 | 5.333 | 5.333 | Fast symbol operations | Magic-mul instead of bits-shift for division |
| Base40               | 5.333 | 5.333 | 5.333 | Fast symbol operations | Same values can be encoded in different ways without strict rules |
| Base64               | 6.000 | 6.000 | 6.000 | Fast symbol operations | Perfectly fits to 192-bits vector |