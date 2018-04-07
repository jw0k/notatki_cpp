# notatki_cpp

## Template template parameters i defaultowe parametry

```cpp
template<typename T, template<typename> class Container>
class Widget {};
```

Do takiej klasy przed C+\+17 nie można było przekazać `std::vector` jako `Container`, bo `std::vector` ma 2 parametry szablonowe (typ elementu i alokator), a nie jeden. Nawet mimo że drugi parametr (alokator) ma wartość domyślną. Od C+\+17 można:

```cpp
Widget<int, std::vector> w; //error przed C++17, OK w C++17
```

Jednak Clang wymaga podania flagi `-frelaxed-template-template-args`, żeby to się skompilowało. Wynika to z tego, że defect report P0522R0, który naprawia CWG 150, sprawia, że kod, który wcześniej nie był ambiguous, teraz jest. Np.:

```cpp
template<template<typename> class>
void foo() {}

template<template<typename, typename> class>
void foo() {}

template<typename, typename = void>
struct Bar {};

int main()
{
    foo<Bar>(); // ambiguous po wprowadzeniu P0522R0; po naprawieniu tekstu w standardzie, powinno być wybrane drugie foo()
}
```

GCC kompiluje `Widget`'a bez żadnych flag. Powyższy kod przestanie się kompilować w GCC po zmianie np. z C+\+14 na C+\+17.


## Pełna specjalizacja szablonowej metody klasy szablonowej

```cpp
template<typename T>
struct Widget
{
    template<typename U>
    void foo(U) {}
};
```

Jak wyspecjalizować `foo` np. dla inta? Nie da się tego zrobić w namespace scope (poza klasą). Zabrania tego wprost standard (§ 17.8.3/17). `Widget` musiałby być też wyspecjalizowany, żeby móc specjalizować `foo`. A więc taki kod jest OK:

```cpp
template<>
template<>
void Widget<int>::foo<double>(double) {}
```
    
Albo jeszcze krócej:

```cpp
template<>
template<>
void Widget<int>::foo(double) {} //można pominąć <double> po foo, bo foo przyjmuje double'a i można ten typ dzięki temu wydedukować
```

Ale nie można zrobić tak:

```cpp
template<typename T>
template<>
void Widget<T>::foo(double) {} //ERROR! Np. w Clangu: "cannot specialize (with 'template<>') a member of an unspecialized template"
```

Pełna specjalizacja w class scope (wewnątrz klasy) powinna być możliwa, ale nie kompiluje tego ani GCC, ani Clang (edit 1.04.2018 - najnowszy Clang - 7.0.0 z HEADa już sobie z tym daje radę), natomiast MSVC już tak. Workaroundem dla GCC/Clang jest wrzucenie `foo` do zagnieżdżonej struktury i częściowa specjalizacja tej struktury. Częściową specjalizację GCC/Clang skompiluje, bo była ona dopuszczona w class scope już w starych wersjach C+\+ - naprawia to CWG 727 (dopuszcza nie tylko częściowe, ale też i pełne specjalizacje w class scope). Przykład workarounda:

```cpp
template<typename T>
struct Widget
{
    template<typename U, typename = void>
    struct FooImpl
    {
        static void foo(U) {}
    };

    template<typename Dummy>
    struct FooImpl<double, Dummy>
    {
        static void foo(double) {}
    };

    template<typename U>
    void foo(U&& u)
    {
        FooImpl<std::decay_t<U>>::foo(std::forward<U>(u));
    }
};
```

## Specjalizacje funkcji nie overloadują

W C+\+ specjalizacje funkcji **nie biorą udziału** w overload resolution. Gdy wołasz funkcję, która ma overloady, najpierw wybierany jest najbardziej pasujący overload (bez patrzenia na żadne specjalizacje), a dopiero potem przeszukiwane są ewentualne specjalizacje i wybierana jest najbardziej pasująca.

Taki kod działa zgodnie z intuicją:

```cpp
template<typename T>
void foo(T);

template<typename T>
void foo(T*);

template<>
void foo(int*);

int main()
{
    int* p = nullptr;
    foo(p); //wybierze foo(int*)
    return 0;
}
```
    
Ale taki już nie:

```cpp
template<typename T>
void foo(T);

template<>
void foo(int*);

template<typename T>
void foo(T*);

int main()
{
    int* p = nullptr;
    foo(p); //wybierze foo(T*) !!!
    return 0;
}
```
    
Dokładniejsze wytłumaczenie: http://www.gotw.ca/publications/mill17.htm


## Ukrywanie nazw:

Funkcja o nazwie `foo` w klasie pochodnej *ukrywa* wszystkie funkcje o nazwie `foo` z klas bazowych. Liczy się tylko nazwa funkcji, nie cała sygnatura.

```cpp
struct Base
{
    void foo() {}
};

struct Derived : Base
{
    void foo(int) {}
};

int main()
{
    Derived d;
    d.foo(); //ERROR! widoczna jest tylko funkcja void foo(int)
    d.Base::foo(); //składnia nieco egzotyczna, ale działa
}
```
    
Rozwiązaniem jest `using Base::foo;` w klasie `Derived`. Uwidacznia to *wszystkie* funkcje o nazwie `foo` z klasy `Base`. W przypadku gdy funkcja `foo` występuje w kilku klasach bazowych, ale nie ma `foo` w klasie pochodnej, wywołanie `foo` jest niejednoznaczne:

```cpp
struct Base1
{
    void foo(int) {}
};

struct Base2
{
    void foo(std::string) {}
};

struct Derived : Base1, Base2
{
};

int main()
{
    Derived d;
    d.foo(5); //ERROR - ambiguous, pomimo że oczywistym wyborem byłoby void foo(int)
}
```
    
W tym przypadku również sprawę załatwia `using`:

```cpp
struct Derived : Base1, Base2
{
    using Base1::foo;
    using Base2::foo;
};
```

## Transparentne komparatory

Od C+\+14 funkcje z posortowanych asocjacyjnych kontenerów (`set`, `map`, `multiset`, `multimap`), które przeszukują te kontenery według klucza (`find`, `count`, `equal_range`, `lower_bound`, `upper_bound`) mogą przyjmować nie tylko klucz, ale też obiekt dowolnego typu, który jest porównywalny z typem klucza. Rozwiązuje to następujący problem:

```cpp
struct Employee
{
    Employee(int id, std::string name): id_(id), name_(name) {}

    int id_;
    std::string name_;
};

bool operator<(Employee const& e1, Employee const& e2)
{
    return e1.id_ < e2.id_;
}

int main()
{  
    std::set<Employee> s;

    s.emplace(2, "janusz");
    s.emplace(4, "john");
    s.emplace(6, "mike");

    //chcemy znaleźć pracownika z id==4; trzeba stworzyć obiekt Employee, mimo, że imię nas nie interesuje!
    //jeśli tworzenie nowego obiektu Employee jest kosztowne, to rozwiązanie jest nieefektywne!
    s.find(Employee{4, ""}); 
}
```

Aby skorzystać z nowych wersji `find`, `count`, itp., należy użyć *transparentnego* komparatora:

```cpp
struct CompareEmployees
{
    using is_transparent = void; // ta deklaracja włącza nowe wersje find, count, itp.

    bool operator()(Employee const& l, Employee const& r) const
    {
        return l.id_ < r.id_;
    }

    bool operator()(int id, Employee const& e) const
    {
        return id < e.id_;
    }

    bool operator()(Employee const& e, int id) const
    {
        return e.id_ < id;
    }
};
```
    
Teraz możliwe jest wyszukiwanie po `id`:

```cpp
int main()
{  
    std::set<Employee, CompareEmployees> s;

    s.emplace(2, "janusz");
    s.emplace(4, "john");
    s.emplace(6, "mike");

    s.find(4); 
}
```
    
Alternatywnym rozwiązaniem jest użycie `std::less<>`, który też jest transparentnym komparatorem:

```cpp
struct Employee
{
    Employee(int id, std::string name): id_(id), name_(name) {}

    int id_;
    std::string name_;
};

bool operator<(Employee const& l, Employee const& r)
{
    return l.id_ < r.id_;
}

bool operator<(int id, Employee const& e)
{
    return id < e.id_;
}

bool operator<(Employee const& e, int id)
{
    return e.id_ < id;
}

int main()
{  
    std::set<Employee, std::less<>> s;

    s.emplace(2, "janusz");
    s.emplace(4, "john");
    s.emplace(6, "mike");

    s.find(4);
}
```

Zdecydowano, że domyślnie ten ficzer jest wyłączony i trzeba go świadomie włączyć (opt-in) używając `std::less<>` albo pisząc swój komparator, w którym istnieje typ `is_transparent` (dowolny, niekoniecznie void). Taka decyzja podyktowana jest tym, że ten ficzer zmienia zachowanie istniejącego kodu, np:

```cpp
std::set<std::string> s = /* ... */;
s.find("key");
```
    
Bez transparentnego komparatora nastąpi tutaj konstrukcja tymczasowego obiektu `std::string` przy pomocy `"key"`. Następnie ten tymczasowy obiekt będzie użyty do wszystkich porównań. Natomiast w przypadku transparentnego komparatora, `find` przekaże `"key"` bez żadnej konwersji do komparatora. Konwersja do `std::string` nastąpi dopiero w komparatorze, wielokrotnie, przy każdym porównaniu. LWG uznało to za poważny problem performance'owy.

Nazwa `is_transparent` nie jest zbyt szczęśliwa, lepszy by była `is_polymorphic` - taki komparator działa dla różnych typów, więc jest to rodzaj polimorfizmu (statycznego). Niestety `is_polymorphic` występuje już w bibliotece standardowej.

## Too perfect forwarding

Gdy zachodzi potrzeba, by nasza klasa miała konstruktor, który przyjmuje pewną wartość i przekazuje dalej jej kategorię wartości (lvalue/rvalue), zazwyczaj używamy perfect forwardingu:

```cpp
struct Widget
{
    Widget() = default;
    
    template<typename T>
    Widget(T&&)
    {
        std::cout << "template constructor\n";
    }
};
```

Rodzi to powien problem: nasz konstruktor może się również odpalić, gdy `T == Widget`, lub `T == Widget&`. Taki szablonowy konstruktor forwardujący nie powoduje, że kompilator przestanie generować zwykły konstruktor kopiujący/przenoszący - `Widget` posiada wygenerowany konstruktor kopiujący: `Widget(Widget const&)`. Konstruktor domyślny jest jednak suppressowany, a ze względu na to, że przyda się w tym przykładzie, został zadeklarowany jako `= default`.

Zasady overload resolution konstruktorów mogą zaskakiwać:

```cpp
Widget w1;
Widget w2(w1); //wywoła konstruktor szablonowy!
```

W powyższym przykładzie **konstruktor szablonowy jest lepszym dopasowaniem**, ponieważ nie wymaga konwersji z `Widget` do `Widget const`, tak jak konstruktor kopiujący. Z kolei `Widget const` **bardziej pasuje do zwykłego konstruktora kopiującego**:

```cpp
Widget const w1;
Widget w2(w1); //wywoła konstruktor kopiujący
```

Jeśli nie chcemy, żeby szablonowy konstruktor był wybierany gdy `T == Widget&`, możemy zadeklarować dwa konstruktory kopiujące: `Widget(Widget const&){}` oraz `Widget(Widget&){}`. Oba z nich są lepszym dopasowaniem niż konstruktor szablonowy. Jednak problem nadal występuje, gdy spróbujemy wywołać konstruktor kopiujący podając mu obiekt klasy, która dziedziczy po `Widget` - wtedy znowu zostanie wybrany konstruktor szablonowy.

Lepszym rozwiązaniem jest ograniczenie typów jakie może przyjmować szablon. Robi się to poprzez `requires` (od C++20) albo SFINAE (np. `std::enable_if`). Przykładowo, jeśli chcemy przyjmować tylko typy stringopodobne:

```cpp
template<typename T, typename = std::enable_if_t<std::is_convertible_v<T,std::string>>>
Widget(T&&)
{
    std::cout << "template constructor\n";
}
```

Czasem jednak zachodzi potrzeba, by mieć szablonowy konstruktor kopiujący, który jest zawsze preferowany. W poniższym przykładzie zwykły konstruktor kopiujący, tak jak poprzednio, będzie zawsze lepszym dopasowaniem niż szablonowy:

```cpp
struct Widget
{
    Widget() = default;
        
    template<typename T>
    Widget(T const&)
    {
        std::cout << "template constructor\n";
    }
};

Widget w1;
Widget w2(w1); //wywoła zwykły konstruktor kopiujący
```

Dodanie `Widget(Widget const&) = delete;` nie jest rozwiązaniem - membery oznaczone jako `delete` nadal uczestniczą w overload resolution, więc `Widget w2(w1)` się nie skompiluje. Problem można rozwiązać przy pomocy triku: dodając `Widget(Widget const volatile&) = delete;` do klasy `Widget`. Ma to dwa skutki: oznacza konstruktor kopiujący przyjmujący `Widget const volatile` jako `delete` oraz, co nas bardziej interesuje, zapobiega wygenerowaniu zwykłego konstruktora kopiującego (nie ma go w zbiorze overloadów):

```cpp
struct Widget
{
    Widget() = default;
        
    template<typename T>
    Widget(T const&)
    {
        std::cout << "template constructor\n";
    }
    
    Widget(Widget const volatile&) = delete;
};

Widget w1;
Widget w2(w1); //wywoła konstruktor szablonowy
```

## Tablice butwieją do prvalue wskaźników

Tablica elementów typu T konwertuje się (butwieje*) do wskaźnika-na-T na pierwszy element. W wyniku tej konwersji powstaje wyrażenie, którego kategorią wartości jest **prvalue**. Przykładowo:

```cpp
void bar(int (*&))
{
}

void bar(int (*&&))
{
}

int a[5];
bar(a); //wywoła bar(int (*&&))
```

Takie samo zachowanie można zaobserwować dla string literali, które są niczym innym jak N-elementową tablicą `char const`'ów, gdzie N jest liczbą znaków plus 1. Warto też wiedzieć, że wyrażenie będące string literalem ma kategorię wartości **lvalue**. Po zbutwieniu staje się jednak prvalue wskaźnikiem.


\*Tak sobie przetłumaczyłem angielskie *decay*. Handluj z tym.
