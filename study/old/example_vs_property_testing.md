# 1. Testowanie na przykładach (Example-Based Testing)
W testowaniu na przykładach sprawdzamy funkcję na podstawie konkretnych wartości wejściowych i oczekiwanych rezultatów.

### Załóżmy, że chcemy przetestować funkcję dodawania.

Testowanie na przykładach:
```fsharp
[<Test>]
let ``Add two numbers, expect their sum``() =
let testData = [ (1,2,3); (2,2,4); (3,5,8); (27,15,42) ]
for (x,y,expected) in testData do
    let actual = add x y
    Assert.AreEqual(expected,actual)
 ```

A deweloper potrafi napisać coś takiego...
```fsharp
let add x y =
    match x,y with
    | 1,2 -> 3
    | 2,2 -> 4
    | 3,5 -> 8
    | 27,15 -> 42
    | _ -> 0
```
Dodatkowo odgrażając się, że podąża za regułą TDD, czyli poprawia swój kod minimalnie dopóki test nie będzie zielony :)

Ograniczenia:

- Musimy ręcznie wymyślić testowe dane wejściowe.
- Testy obejmują tylko przypadki, które sami zdefiniujemy.
- Trudno przewidzieć wszystkie możliwe scenariusze (np. wartości graniczne, liczby ujemne, liczby bardzo duże itp.).

Można wymusić prawidłowy kod na podstawie generatora liczb losowych:
```fsharp
[<Test>]
let ``Add two random numbers 100 times, expect their sum``() =
for _ in [1..100] do
    let x = randInt()
    let y = randInt()
    let expected = x + y
    let actual = add x y
    Assert.AreEqual(expected,actual)
```
Natomiast w tym przypadku musimy zaimplementować w teście ```add```, aby przetestować funkcję ```add```

# 2. Testowanie na podstawie właściwości (Property-Based Testing)

Zamiast testować pojedyncze przypadki, definiujemy ogólne właściwości, które funkcja powinna zawsze spełniać. Natomiast są różne narzędzia, np. FsCheck, który automatycznie generuje dane wejściowe i sprawdza, czy właściwości są spełniane.

Przykładowo, dla funkcji dodawania możemy zdefiniować następujące właściwości:
 - przemienność
 - dodawanie dwukrotne 1 jest tym samym co jednokrotne dodanie 2
 - dodanie do liczby 0 nie zmienia wartości liczby

 ```fsharp
let add x y = x + y

let commutativeProperty x y =
  let result1 = add x y
  let result2 = add y x 
  result1 = result2

[<Test>]
let addDoesNotDependOnParameterOrder() =
  Check.Quick commutativeProperty

let add1TwiceIsAdd2Property x _ =
  let result1 = x |> add 1 |> add 1
  let result2 = x |> add 2
  result1 = result2

[<Test>]
let addOneTwiceIsSameAsAddTwo() =
  Check.Quick add1TwiceIsAdd2Property

let identityProperty x _ =
  let result1 = x |> add 0
  result1 = x

[<Test>]
let addZeroIsSameAsDoingNothing() =
  Check.Quick identityProperty
 ```
Dzięki podejściu ```property-based``` mamy większą pewność, że implementacja jest właściwa. Dodatkowo to podejście pozwala zrozumieć lepiej wymagania i istotę całego problemu.\
Zdefiniowanie właściwości w bardziej złożonych przypadkach potrafi być bardzo problematyczne, natomiast istnieją pewne techniki, które pomagają nam w ich definiowaniu.