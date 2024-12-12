# 1. QuickCheck

Został stworzony w języku Haskell jako pierwsze narzędzie wspierające testowanie oparte na właściwościach. Jest on inspiracją dla bibliotek w innych językach, takich jak FsCheck dla .NET.

Cechy QuickCheck:
- Automatyczne generowanie danych testowych.
- Sprawdzanie, czy zdefiniowane właściwości funkcji są spełniane dla wielu losowych przypadków.
- W przypadku wykrycia błędu, narzędzie redukuje (ang. shrinking) dane wejściowe, aby znaleźć minimalny przykład prowadzący do błędu.

![image](schema.png)

## Działanie

1. **Checker API** wykrywa typ wejścia funkcji.
2. Wywoływany jest generator odpowiedniego typu.
3. Następuje generowanie przypadków testowych.
4. Przypadki testowe są przekazywane do testowanej właściwości. 

## Funkcje

- **Check.Quick** – uruchamia szybki test, sprawdzając właściwość dla domyślnej liczby losowych przypadków (np. 100).
- **Check.Verbose** – działa jak Check.Quick, ale wyświetla więcej szczegółowych informacji o danych testowych.
- **Generowanie danych wejściowych** – FsCheck wspiera generowanie danych dla typów prostych (np. int, float, string) i bardziej złożonych struktur, takich jak listy czy rekordy.

## Pisanie właściwości w FsCheck

FsCheck umożliwia definiowanie właściwości w formie funkcji logicznych. Właściwości opisują, jakie warunki zawsze muszą być spełnione przez funkcję.

Przykład: Dla funkcji reverse, która odwraca listę, możemy zdefiniować właściwość:
- Odwrócenie listy dwukrotnie powinno zwrócić pierwotną listę.

```fsharp
let reverseProperty (xs: int list) =
    List.rev (List.rev xs) = xs

Check.Quick reverseProperty
```

## Generacja danych testowych - arbitraż
FsCheck pozwala na tworzenie własnych generatorów danych wybranego typu. Przydatne, gdy chcemy testować funkcję na specyficznych danych.
Na początku tworzymy generator, następnie generujemy dane testowe podając maksymalny rozmiar przypadku testowego (w przypadku stuktur danych) oraz liczbę przypadków testowych.

```fsharp
let intGenerator = Arb.generate<int>
Gen.sample 100 3 intGenerator  // [-37; 24; -62]

//Przykład arbitrażu dla liczb dodatnich:
type PositiveInt = PositiveInt of int

let positiveIntGen =
    Gen.suchThat ((<) 0) Arb.generate<int>
Arb.register<PositiveInt>(positiveIntGen)

type Generators =
    static member PositiveInt() =
        Arb.fromGen positiveIntGen
Arb.register<Generators>() // rejestracja generatora by Quick.Check mógł zostać użyty

let intListGenerator = Arb.generate<int list>
Gen.sample 5 10 intListGenerator // [ []; []; [-4]; [0; 3; -1; 2]; [1]; [1]; []; [0; 1; -2]; []; [-1; -2]]

let stringGenerator = Arb.generate<string>
Gen.sample 10 3 stringGenerator // [""; "eiX$a^"; "U%0Ika&r"]

type Point = {x:int; y:int; color: Color}
let pointGenerator = Arb.generate<Point>
Gen.sample 50 10 pointGenerator
(*
[{x = -8; y = 12; color = Green -4;};
  {x = 28; y = -31; color = Green -6;};
  {x = 11; y = 27; color = Red;};
  {x = -2; y = -13; color = Red;};
  {x = 6; y = 12; color = Red;};
  // itd
*)
```
## Shrinking

Po fazie generowania danych losowych danych wejściowych, są one ustawione od najmniejszej do najwiekszej wartości.\
Jeśli jakiekolwiek dane wejściowe spowodują, że właściwość przestanie być spełniona, narzędzie zaczyna "zmniejszać" pierwszy parametr, aby znaleźć mniejszą wartość. Dokładny proces zmniejszania zależy od typu danych (można go również nadpisać) *(W przypadku liczb prowadząc do coraz mniejszych wartości.)*\

```fsharp
let isSmallerThan80 x = x < 80
isSmallerThan80 100 // false, so start shrinking

Arb.shrink 100 |> Seq.toList//  [0; 50; 75; 83; 94; 97; 99]
isSmallerThan80 0 // true
isSmallerThan80 50 // true
isSmallerThan80 75 // true
isSmallerThan80 83 // false, so shrink again

Arb.shrink 83 |> Seq.toList//  [0; 44; 66; 77; 80; 81; 83]
isSmallerThan80 0 // true
isSmallerThan80 44 // true
isSmallerThan80 66 // true
isSmallerThan80 77 // true
isSmallerThan80 80 // false <- najmniejsza porażka
// wynik: Falsifiable, after 10 tests (2 shrinks)
```
Narzędzie jest bardzo przydatne do określenia, gdzie znajdują się granice błędów w testowaniu\
Shrink działa na customowych typach złożonych, dodatkowo można też generować własne sekwencje oraz zasady w jaki sposób przeprowadzać customowe shrinkowanie.

```fsharp
Arb.shrink "abcd" |> Seq.toList
//  ["bcd"; "acd"; "abd"; "abc"; "abca";
//   "abcb"; "abcc"; "abad"; "abbd"; "aacd"]
```

## Konfiguracja
Czasem może zaistnieć potrzeba własnego dostosowania liczby testów itp. W tym celu można odpowiednio skonfigurować narzędzie:
```fsharp
let config = {
  Config.Quick with
    MaxTest = 1000
  }
Check.One(config,isSmallerThan80 )
// result: Ok, passed 1000 tests. (a nie powinno :)

let config = {
  Config.Quick with
    MaxTest = 10000
  }
Check.One(config,isSmallerThan80 )
// result: Falsifiable, after 8660 tests (1 shrink):
//         80
```
## Warunki wstępne
```fsharp
let preCondition x y =
  (x,y) <> (0,0)
  && (x,y) <> (2,2)

let additionIsNotMultiplication_withPreCondition x y =
  preCondition x y ==> additionIsNotMultiplication x y
  // Ok, passed 100 tests.
```
Jak widać, tego rodzaju rozwiązania mogą być użyte tylko w nielicznych przypadkach, gdy możemy zdefiniować niewielką liczbę "wyjątków od reguły"

## Testowanie kilku właściwości na raz
```fsharp
type AdditionSpecification =
  static member ``Commutative`` x y =
    commutativeProperty x y
  static member ``Associative`` x y z =
    associativeProperty x y z
  static member ``Left Identity`` x =
    leftIdentityProperty x
  static member ``Right Identity`` x =
    rightIdentityProperty x

Check.QuickAll<AdditionSpecification>()

--- Checking AdditionSpecification ---
AdditionSpecification.Commutative-Ok, passed 100 tests.
AdditionSpecification.Associative-Ok, passed 100 tests.
AdditionSpecification.Left Identity-Ok, passed 100 tests.
AdditionSpecification.Right Identity-Ok, passed 100 tests.
```
