# notatki_cpp

## Template template parameters i defaultowe parametry

    template<typename T, template<typename> class Container>
    class Widget {};

Do takiej klasy przed C+\+17 nie można było przekazać `std::vector` jako `Container`, bo `std::vector` ma 2 parametry szablonowe (typ elementu i alokator), a nie jeden. Nawet mimo że drugi parametr (alokator) ma wartość domyślną. Od C+\+17 można:

    Widget<int, std::vector> w; //error przed C++17, OK w C++17

Jednak Clang wymaga podania flagi `-frelaxed-template-template-args`, żeby to się skompilowało. Wynika to z tego, że defect report P0522R0, który naprawia CWG 150, sprawia, że kod, który wcześniej nie był ambiguous, teraz jest. Np.:

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

GCC kompiluje `Widget`'a bez żadnych flag. Powyższy kod przestanie się kompilować w GCC po zmianie np. z C+\+14 na C+\+17.


## Pełna specjalizacja szablonowej metody klasy szablonowej

    template<typename T>
    struct Widget
    {
        template<typename U>
        void foo(U) {}
    };

Jak wyspecjalizować `foo` np. dla inta? Nie da się tego zrobić w namespace scope (poza klasą). Zabrania tego wprost standard (§ 17.8.3/17). `Widget` musiałby być też wyspecjalizowany, żeby móc specjalizować `foo`. A więc taki kod jest OK:

    template<>
    template<>
    void Widget<int>::foo<double>(double) {}
    
Albo jeszcze krócej:

    template<>
    template<>
    void Widget<int>::foo(double) {} //można pominąć <double> po foo, bo foo przyjmuje double'a i można ten typ dzięki temu wydedukować

Ale nie można zrobić tak:

    template<typename T>
    template<>
    void Widget<T>::foo(double) {} //ERROR! Np. w Clangu: "cannot specialize (with 'template<>') a member of an unspecialized template"

Pełna specjalizacja w class scope (wewnątrz klasy) powinna być możliwa, ale nie kompiluje tego ani GCC, ani Clang, natomiast MSVC już tak. Workaroundem dla GCC/Clang jest wrzucenie `foo` do zagnieżdżonej struktury i częściowa specjalizacja tej struktury. Częściową specjalizację GCC/Clang skompiluje, bo była ona dopuszczona w class scope już w starych wersjach C+\+ - naprawia to CWG 727 (dopuszcza nie tylko częściowe, ale też i pełne specjalizacje w class scope). Przykład workarounda:

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


## Specjalizacje funkcji nie overloadują

W C+\+ specjalizacje funkcji **nie biorą udziału** w overload resolution. Gdy wołasz funkcję, która ma overloady, najpierw wybierany jest najbardziej pasujący overload (bez patrzenia na żadne specjalizacje), a dopiero potem przeszukiwane są ewentualne specjalizacje i wybierana jest najbardziej pasująca.

Taki kod działa zgodnie z intuicją:

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
    
Ale taki już nie:

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
    
Dokładniejsze wytłumaczenie: http://www.gotw.ca/publications/mill17.htm


## Ukrywanie nazw:

Funkcja o nazwie `foo` w klasie pochodnej *ukrywa* wszystkie funkcje o nazwie `foo` z klas bazowych. Liczy się tylko nazwa funkcji, nie cała sygnatura.

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
    
Rozwiązaniem jest `using Base::foo;` w klasie `Derived`. Uwidacznia to *wszystkie* funkcje o nazwie `foo` z klasy `Base`. W przypadku gdy funkcja `foo` występuje w kilku klasach bazowych, ale nie ma `foo` w klasie pochodnej, wywołanie `foo` jest niejednoznaczne:

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
    
W tym przypadku również sprawę załatwia `using`:

    struct Derived : Base1, Base2
    {
        using Base1::foo;
        using Base2::foo;
    };


## Transparentne komparatory

Od C+\+14 funkcje z posortowanych asocjacyjnych kontenerów (`set`, `map`, `multiset`, `multimap`), które przeszukują te kontenery według klucza (`find`, `count`, `equal_range`, `lower_bound`, `upper_bound`) mogą przyjmować nie tylko klucz, ale też obiekt dowolnego typu, który jest porównywalny z typem klucza. Rozwiązuje to następujący problem:

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

Aby skorzystać z nowych wersji `find`, `count`, itp., należy użyć *transparentnego* komparatora:

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
    
Teraz możliwe jest wyszukiwanie po `id`:

    int main()
    {  
        std::set<Employee, CompareEmployees> s;
        
        s.emplace(2, "janusz");
        s.emplace(4, "john");
        s.emplace(6, "mike");
        
        s.find(4); 
    }
    
Alternatywnym rozwiązaniem jest użycie `std::less<>`, który też jest transparentnym komparatorem:

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

Zdecydowano, że domyślnie ten ficzer jeset wyłączony i trzeba go świadomie włączyć (opt-in) używając `std::less<>` albo pisząc swój komparator, w którym istnieje typ `is_transparent` (dowolny, niekoniecznie void). Taka decyzja podyktowana jest tym, że ten ficzer zmienia zachowanie istniejącego kodu, np:

    std::set<std::string> s = /* ... */;
    s.find("key");
    
Bez transparentnego komparatora nastąpi tutaj konstrukcja tymczasowego obiektu `std::string` przy pomocy `"key"`. Następnie ten tymczasowy obiekt będzie użyty do wszystkich porównań. Natomiast w przypadku transparentnego komparatora, `find` przekaże `"key"` bez żadnej konwersji do komparatora. Konwersja do `std::string` nastąpi dopiero w komparatorze, wielokrotnie, przy każdym porównaniu. LWG uznało to za poważny problem performance'owy.

Nazwa `is_transparent` nie jest zbyt szczęśliwa, lepszy by była `is_polymorphic` - taki komparator działa dla różnych typów, więc jest to rodzaj polimorfizmu (statycznego). Niestety `is_polymorphic` występuje już w bibliotece standardowej.
