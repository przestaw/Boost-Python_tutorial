# Boost.Python

## Wymagania

- interpreter Pythona
- pliki nagłówkowe Pythona (pakiet deweloperski)
- biblioteka Boost.Python

## Pierwsze kroki

Zacznijmy od minimalnego działającego przykładu — funkcji obliczającej kwadrat liczby.

1. Napiszmy kod w C++:

```cpp
#include <boost/python.hpp>

using namespace boost::python;

// najpierw definiujemy funkcję — w tym przypadku square
int square( int a ) {
	return a*a;
}

// następnie tworzymy moduł o nazwie "example"
BOOST_PYTHON_MODULE(example) {

	// na koniec udostępniamy funkcję w module "example"
	def("square", square, "Returns a square of given number");
}
```

2. Skompilujmy kod jako bibliotekę współdzieloną — `*.so` w systemach uniksowych oraz `*.dll` (z rozszerzeniem `*.pyd` dla modułów Pythona) w systemie Windows.

3. Wynikowa biblioteka współdzielona jest teraz widoczna z poziomu Pythona. Przykładowa sesja interaktywna:

```python
python3
>>> import example
>>> print(example.square(2))
4
```

## Udostępnianie klas i funkcji

Za pomocą Boost.Python możemy udostępniać w Pythonie zarówno klasy, jak i funkcje.
Każdy z tych elementów może posiadać osobny ciąg dokumentujący (docstring), przekazywany jako ostatni argument.

### Udostępnianie funkcji

Aby udostępnić funkcję, używamy wywołania funkcji `def`.
Nadajemy funkcji nazwę widoczną w Pythonie, która może, ale nie musi, pokrywać się z jej nazwą w kodzie źródłowym C++.
Musimy przekazać wskaźnik na funkcję, aby powiązać ją ze wskazaną nazwą.
Dobrą praktyką jest opatrzenie każdej udostępnianej funkcji krótką dokumentacją (docstringiem).

```cpp
template <class Fn>
void def(char const* name, Fn fn);

template <class Fn, class A1>
void def(char const* name, Fn fn, A1 const&);

template <class Fn, class A1, class A2>
void def(char const* name, Fn fn, A1 const&, A2 const&);

template <class Fn, class A1, class A2, class A3>
void def(char const* name, Fn fn, A1 const&, A2 const&, A3 const&);
```

Funkcje z rodziny `def` przyjmują od 2 do 5 argumentów:
- `name` — nazwa funkcji zgodna z konwencją nazewniczą Pythona,
- `fn` — wskaźnik na funkcję C++,
- `a1`–`a3` — docstring, argumenty nazwane (keywords) lub polityki wywołań (`CallPolicy`) w dowolnej kolejności.

Docstring to zwięzły opis funkcji, analogiczny do napisu dokumentującego umieszczanego w pierwszej linii funkcji w Pythonie.
Obiekt argumentów nazwanych (keywords) przechowuje sekwencję nazw parametrów, a jego typ koduje liczbę zdefiniowanych argumentów nazwanych. Wyrażenie opisujące argumenty nazwane może również określać wartości domyślne dla wybranych lub wszystkich parametrów.

Przykład użycia argumentów nazwanych i wartości domyślnych:

```cpp
#include <boost/python/def.hpp>
using namespace boost::python;

int foo(double x, double y, double z=0.0, double w=1.0);

BOOST_PYTHON_MODULE(xxx)
{
   def("foo", foo
            , ( arg("x"), "y", arg("z")=0.0, arg("w")=1.0 )
            );
}
```

Kolejny przykład z wykorzystaniem funkcji `square`:

```cpp
// udostępnienie funkcji myNamespace::square pod nazwą "squareInt"
// pierwszy argument to nazwa funkcji w Pythonie
// drugi argument to wskaźnik na funkcję C++
// trzeci (opcjonalny) argument to dokumentacja (docstring) w module
// czwarty (opcjonalny) argument to nazwa parametru
def("squareInt", &myNamespace::square, "Returns a square of given number", arg("number"));
```

Możemy również udostępnić konkretną specjalizację szablonu funkcji jako funkcję Pythona.
Rozważmy szablon funkcji `square`:

```cpp
template<typename T>
T square( T val ) {
	return val*val;
}
```

Możemy wystawić go zarówno dla typu całkowitoliczbowego, jak i zmiennoprzecinkowego, nadając im osobne nazwy w Pythonie:

```cpp
// udostępnienie specjalizacji square dla liczb zmiennoprzecinkowych
def("squareFloat", square<double>, "Returns a square of given number");
// udostępnienie specjalizacji square dla liczb całkowitych
def("squareInt", square<int>, "Returns a square of given number");
```

### Udostępnianie klas

Do udostępniania klas służą obiekty szablonu `class_<Type>`, dostarczające metody do deklarowania właściwości, metod oraz pól składowych.
Rozważmy podstawowy przykład:

```cpp
#include <boost/python.hpp>
using namespace boost::python;

struct World {
	void set(std::string msg) { this->msg = msg; }
	std::string greet() { return msg; }
	std::string msg;
};

BOOST_PYTHON_MODULE(hello) {
	class_<World>("World")
		.def("greet", &World::greet)
		.def("set", &World::set)
  	;
}
```

Przykładowa sesja w Pythonie:

```python
>>> import hello
>>> planet = hello.World()
>>> planet.set('bioweb')
>>> planet.greet()
'bioweb'
```

Każda z metod `class_<Type>` zwraca referencję do samego obiektu (`*this`), co pozwala na wygodne łańcuchowe łączenie wywołań (method chaining).
W powyższym przykładzie użyliśmy metody `def`, która ma identyczną składnię jak funkcja wolnostojąca służąca do rejestrowania funkcji modułu.

#### Konstruktory

Do definiowania konstruktorów wykorzystujemy tzw. wyrażenia inicjalizujące (`init`-expressions).
Mają one następującą postać:

```cpp
template <T1 = unspecified,...Tn = unspecified>
struct init {
  	init(char const* doc = 0);
  	template <class Keywords>
	init(Keywords const& kw, char const* doc = 0);
  	template <class Keywords>
	init(char const* doc, Keywords const& kw);
};
```

Gdzie `Tn` oznacza typ n-tego argumentu konstruktora, a `Keywords` reprezentuje powiązania nazw argumentów.
Aby udostępnić konstruktor dla klasy, przekazujemy wyrażenie `init<...>` bezpośrednio do konstruktora `class_`.
Jeśli klasa posiada więcej niż jeden konstruktor, kolejne przeciążenia rejestrujemy za pomocą metody `def`:

```cpp
template <class Init>
class_& def(Init init_expr);
```

Jeżeli w ogóle nie chcemy udostępniać konstruktora po stronie Pythona, możemy przekazać znacznik `no_init`.
Dodatkowo, gdy udostępniana klasa jest abstrakcyjna (posiada metody czysto wirtualne), musimy poinformować bibliotekę, że klasa jest niekopiowalna (`boost::noncopyable`). Domyślnie Boost.Python próbuje zarejestrować konwerter dla funkcji zwracających obiekty danej klasy przez wartość. Wymaga to możliwości skopiowania zwracanego obiektu C++ do pamięci zarządzanej przez obiekt Pythona — dla klasy abstrakcyjnej taka próba zakończyłaby się błędem kompilacji.

```cpp
class_<Abstract, boost::noncopyable>("Abstract", no_init)
```

Przeanalizujmy to na przykładach:

- Przykład 1 — dodanie konstruktora do klasy `World`:

```cpp
#include <boost/python.hpp>
using namespace boost::python;

struct World {
	World(std::string msg): msg(msg) {} // dodany konstruktor
  	void set(std::string msg) { this->msg = msg; }
  	std::string greet() { return msg; }
  	std::string msg;
};

BOOST_PYTHON_MODULE(hello) {
	class_<World>("World", init<std::string>(args("msg"), "Constructor for class World"))
		.def("greet", &World::greet)
		.def("set", &World::set)
	;
}
```

- Przykład 2 — klasa z trzema konstruktorami:

```cpp
#include <boost/python.hpp>
using namespace boost::python;

struct Point {
	Point() : x(0), y(0) {}
	Point(int val) : x(val), y(val) {}
	Point(int x, int y) : x(x), y(y) {}
	int x;
	int y;
};

BOOST_PYTHON_MODULE(hello) {
	class_<Point>("Point", init<>("Default constructor"))
		.def(init<int>(args("val"), "Constructor for diagonal points"))
		.def(init<int, int>(args("x", "y"), "Constructor for class point"))
		.def_readwrite("x", &Point::x)
		.def_readwrite("y", &Point::y)
  	;
}
```

#### Metody

Definiowanie metod odbywa się przy użyciu metod z rodziny `def` oraz metody `staticmethod`, która oznacza wskazaną metodę jako statyczną.
Składnia i działanie są analogiczne do rejestrowania zwykłych funkcji, z tą różnicą, że wywołania `def` łączymy łańcuchowo na obiekcie `class_`.

```cpp
#include <boost/python.hpp>
using namespace boost::python;

class class_ : public object {
	...
    // definiowanie metod
    template <class F>
    class_& def(char const* name, F f);
    template <class Fn, class A1>
    class_& def(char const* name, Fn fn, A1 const&);
    template <class Fn, class A1, class A2>
    class_& def(char const* name, Fn fn, A1 const&, A2 const&);
    template <class Fn, class A1, class A2, class A3>
    class_& def(char const* name, Fn fn, A1 const&, A2 const&, A3 const&);

    // deklarowanie metody jako statycznej
    class_& staticmethod(char const* name);
    ...
};
```

Prosty przykład definiowania metod:

```cpp
#include <boost/python.hpp>
using namespace boost::python;

// Prosta klasa
class hello {
    public:
        hello(const std::string& country) { this->country = country; }
        std::string greet() const { return "Hello from " + country; }
    private:
        std::string country;
};

// Funkcja przyjmująca obiekt klasy hello jako argument
std::string invite(const hello& w) {
    return w.greet() + "! Please come soon!";
}

BOOST_PYTHON_MODULE(hello) {
    class_<hello>("hello", init<std::string>())
        .def("greet", &hello::greet)  // dodanie zwykłej metody składowej
        .def("invite", invite)        // dodanie funkcji invite() jako metody klasy
    ;

    def("invite", invite); // invite() może być jednocześnie wolną funkcją modułu
}
```

Tak zdefiniowanych metod i funkcji możemy używać w Pythonie w następujący sposób:

```python
>>> from getting_started2 import *
>>> hi = hello('Poland')
>>> hi.greet()
'Hello from Poland'
>>> invite(hi)
'Hello from Poland! Please come soon!'
>>> hi.invite()
'Hello from Poland! Please come soon!'
```

#### Pola danych i właściwości (properties)

Pola składowe klasy mogą być udostępnione w Pythonie jako atrybuty odpowiadającej jej klasy Pythonowej. Każde udostępniane pole może być tylko do odczytu (read-only) lub do odczytu i zapisu (read-write). Zwykłe oraz statyczne pola danych rejestruje się metodami `def_readonly` oraz `def_readwrite`:

```cpp
class class_ : public object {
	...
	// udostępnianie pól składowych instancji
	template <class D>
	class_& def_readonly(char const* name, D T::*pm);
	template <class D>
	class_& def_readwrite(char const* name, D T::*pm);
	// udostępnianie statycznych pól składowych
	template <class D>
	class_& def_readonly(char const* name, D const& d);
	template <class D>
	class_& def_readwrite(char const* name, D& d);
	...
};
```

Rozważmy przykład klasy `Point` z polami `x` i `y`, zliczającej swoje aktywne instancje:

```cpp
#include <boost/python.hpp>
using namespace boost::python;

struct Point {
	Point(int x, int y) : x(x), y(y) { ++count; }
	~Point() { --count; }
	int x;
	int y;
	static size_t count;
};

size_t Point::count = 0;

BOOST_PYTHON_MODULE(hello) {
	class_<Point>("Point", init<int, int>(args("x", "y"), "Constructor for class point"))
		.def_readwrite("x", &Point::x)
		.def_readwrite("y", &Point::y)
		.def_readonly("pointsCount", &Point::count)
  	;
}
```

W języku C++ klasy z publicznymi polami danych są zazwyczaj źle widziane.
Dobrze zaprojektowane klasy wykorzystujące hermetyzację ukrywają swoje pola prywatne, a dostęp do stanu obiektu odbywa się wyłącznie poprzez metody dostępowe (gettery i settery).
W Pythonie bezpośredni dostęp do atrybutów jest natomiast naturalny i nie narusza hermetyzacji, ponieważ odczyt lub zapis atrybutu może pod spodem wywoływać metody dostępowe (mechanizm `property`).

```cpp
class class_ : public object {
	...
  	// tworzenie właściwości (property) instancji
  	template <class Get>
  	void add_property(char const* name, Get const& fget, char const* doc=0);
  	template <class Get, class Set>
  	void add_property(
	char const* name, Get const& fget, Set const& fset, char const* doc=0);
  	// tworzenie właściwości statycznej
  	template <class Get>
	void add_static_property(char const* name, Get const& fget);
	template <class Get, class Set>
	void add_static_property(char const* name, Get const& fget, Set const& fset);
	...
};
```

Zobaczmy to na przykładzie:

```cpp
#include <boost/python.hpp>
using namespace boost::python;

class Point {
public:
	Point(int x, int y) : x(x), y(y) { ++count; }
	~Point() { --count; }

	int getX() const { return x; }
	int getY() const { return y; }
	void setX(int val) { x = val; }
	void setY(int val) { y = val; }

	static size_t getCount() { return count; }
private:
	int x;
	int y;
	static size_t count;
};

size_t Point::count = 0;

BOOST_PYTHON_MODULE(hello) {
	class_<Point>("Point", init<int, int>(args("x", "y"), "Constructor for class point"))
		.add_property("readOnlyX", &Point::getX, "Read only property X")
		.add_property("x", &Point::getX, &Point::setX, "Property X")
		.add_property("readOnlyY", &Point::getY, "Read only property Y")
		.add_property("y", &Point::getY, &Point::setY, "Property Y")
		.add_static_property("pointsCount", &Point::getCount)
  	;
}
```

#### Dziedziczenie

Klasy rozszerzeń Boost.Python wspierają dziedziczenie pojedyncze oraz wielobazowe w Pythonie, dokładnie tak jak natywne klasy Pythona.
W krotce klas bazowych klasy pochodnej możemy swobodnie łączyć wbudowane klasy Pythona z klasami rozszerzeń C++.
Ilekroć klasa rozszerzenia Boost.Python znajduje się wśród klas bazowych nowej klasy definiowanej w Pythonie, wynikiem również jest klasa rozszerzenia.
W definicji modułu informację o hierarchii dziedziczenia przekazujemy do szablonu `class_` za pomocą parametru szablonu `bases<...>`.

Przykład:

```cpp
struct Base { virtual ~Base(); };
struct Derived : Base {};

void b(Base*);
void d(Derived*);
Base* factory() { return new Derived; }

BOOST_PYTHON_MODULE(inheritance) {
    class_<Base>("Base")
    /*...*/
    ;
    class_<Derived, bases<Base> >("Derived")
    /*...*/
    ;
    def("b", b);
    def("d", d);
    // Poinformuj Pythona o przejęciu własności nad obiektem zwróconym przez fabrykę
    def("factory", factory, return_value_policy<manage_new_object>());
}
```

#### Funkcje wirtualne

Klasy udostępniane przez Boost.Python mogą zachowywać się polimorficznie dzięki funkcjom wirtualnym.
Rozważmy klasę `Base` rozwijającą poprzedni przykład:

```cpp
struct Base {
    virtual ~Base() {}
    virtual int f() = 0;
};
```

Jednym z założeń projektowych Boost.Python jest minimalna ingerencja w istniejący kod C++.
Z założenia powinno być możliwe wystawienie interfejsu zewnętrznej biblioteki bez modyfikowania jej nagłówków.
Nie chcemy dodawać żadnych elementów pomocniczych bezpośrednio do klasy `Base`.
Gdy jednak mamy funkcję wirtualną, która ma być nadpisywana w Pythonie i wywoływana polimorficznie z poziomu C++, potrzebujemy cienkiej warstwy pośredniczącej (wrappera).
Tworzymy więc klasę opakowującą dziedziczącą po `Base`, która bezinwazyjnie przechwytuje wywołania metod wirtualnych i przekazuje je do nadpisanej metody w Pythonie:

```cpp
struct BaseWrap : Base, wrapper<Base> {
    int f() {
        return this->get_override("f")();
    }
};
```

Zwróćmy uwagę, że oprócz dziedziczenia po `Base` zastosowaliśmy dziedziczenie wielobazowe z szablonem `wrapper<Base>`.
Szablon `wrapper` znacząco upraszcza tworzenie klas przeznaczonych do nadpisywania po stronie Pythona.
Fakt, że metoda jest czysto wirtualna, sygnalizujemy za pomocą funkcji `pure_virtual`:

```cpp
class_<BaseWrap, boost::noncopyable>("Base")
    .def("f", pure_virtual(&Base::f))
;
```

Co jednak w sytuacji, gdy metoda `f()` nie jest czysto wirtualna i posiada domyślną implementację?

```cpp
struct Base {
    virtual ~Base() {}
    virtual int f() { return 0; }
};
```

Musimy wówczas rozbudować nasz wrapper tak, aby obsługiwał również domyślną implementację z klasy bazowej:

```cpp
struct BaseWrap : Base, wrapper<Base>
{
    int f()
    {
        if (override f = this->get_override("f")) {
            return f();
        } else {
            return Base::f();
        }
    }

    int default_f() { return this->Base::f(); }
};
```

Zauważmy, że w definicji klasy rejestrujemy zarówno `&Base::f`, jak i `&BaseWrap::default_f`.
Boost.Python musi śledzić dwa elementy:
1. funkcję dyspozytora polimorficznego `f`,
2. funkcję przekazującą wywołanie bezpośrednio do domyślnej implementacji `default_f` (gdy klasa pochodna w Pythonie nie nadpisze metody).

Służy do tego dedykowane przeciążenie metody `def`:

```cpp
class_<BaseWrap, boost::noncopyable>("Base")
    .def("f", &Base::f, &BaseWrap::default_f)
;
```

W Pythonie uzyskamy oczekiwane zachowanie polimorficzne:

```python
>>> base = Base()
>>> class Derived(Base):
...     def f(self):
...         return 42
...
>>> derived = Derived()
>>> base.f()
0
>>> derived.f()
42
```

### Polityki wywołań (`CallPolicy`)

Mechanizm `CallPolicy` pozwala bibliotece Boost.Python bezpiecznie operować na surowych referencjach i wskaźnikach C++.
Poszczególne polityki definiują odmienne strategie zarządzania czasem życia oraz własnością obiektów na granicy C++ i Pythona.

#### `return_internal_reference`

Buduje obiekt Pythona wokół wskaźnika do obiektu wynikowego C++ (dla którego musi istnieć zarejestrowany wrapper `class_<>`) i zarządza czasem życia w taki sposób, aby obiekt nadrzędny (`self`) pozostał przy życiu co najmniej tak długo, jak długo istnieje zwrócony obiekt w Pythonie. Wskaźnik `NULL` (`nullptr`) jest konwertowany na wartość `None`.

#### `return_value_policy<reference_existing_object>`

Podejście naiwne (i potencjalnie niebezpieczne).

Podczas wywołania opakowanej funkcji wartość wskazywana przez zwracaną referencję lub wskaźnik nie jest kopiowana.
Tworzony jest nowy obiekt Pythona przechowujący niezarządzany wskaźnik `U*` do obiektu docelowego, bez żadnej gwarancji, że czas życia obiektu C++ będzie co najmniej tak długi jak czas życia odpowiadającego mu obiektu Pythona.

Klasa ta jest wykorzystywana wewnętrznie w implementacji `return_internal_reference`. Wskaźnik `NULL` również jest zwracany jako `None`.

#### `return_value_policy<manage_new_object>`

Służy do opakowywania funkcji C++ zwracających wskaźnik na obiekt zaalokowany na stercie za pomocą operatora `new`, które oczekują, że wywołujący przejmie odpowiedzialność za zwolnienie tej pamięci (`delete`).
Boost.Python automatycznie usunie obiekt C++ w momencie niszczenia obiektu Pythona przez odświeżacz pamięci.

```cpp
T* factory() { return new T(); }

class_<T>("T");

def("Tfactory", factory, return_value_policy<manage_new_object>() );
```

#### `with_custodian_and_ward<M,N>`

Utrzymuje przy życiu N-ty argument wywołania tak długo, jak długo żyje argument M-ty (opiekun — *custodian*).

Znaczenie parametrów szablonu `M`, `N`:

    1 - pierwszy argument (w przypadku wywołania metody jest to obiekt self)
    2 - drugi argument (pierwszy jawny parametr metody)
    ...

Przykładowo operacja dodawania elementu do kontenera (`append`) zazwyczaj korzysta z `with_custodian_and_ward<1,2>`, co oznacza utrzymanie wstawionego argumentu przy życiu dopóki istnieje sam kontener.

#### `with_custodian_and_ward_postcall<M,N>`

Wiąże czasy życia argumentów oraz wartości zwracanej już po wykonaniu wywołania C++.

Indeksy `M`, `N` działają tak samo jak wyżej, z tą różnicą, że dostępny jest również indeks `0` oznaczający obiekt wynikowy funkcji.

#### Pamięć i inteligentne wskaźniki

Ponieważ Python zarządza alokacją pamięci i odśmiecaniem automatycznie, samo pojęcie „wskaźnika” nie ma w Pythonie bezpośredniego odpowiednika.
Wiele interfejsów API w C++ operuje jednak na surowych wskaźnikach lub inteligentnych wskaźnikach (smart pointers).

##### Surowe wskaźniki

Czasem życia obiektów C++ tworzonych wyrażeniem `new A` może zarządzać mechanizm garbage collectora Pythona przy użyciu polityki `manage_new_object`:

```cpp
struct A {
    static A*   create () { return new A; }
    std::string hello  () { return "Hello, is there anybody in there?"; }
};

BOOST_PYTHON_MODULE(pointer) {
    class_<A>("A", no_init)
        .def("create", &A::create, return_value_policy<manage_new_object>())
        .staticmethod("create")
        .def("hello", &A::hello)
	;
}
```

#### Inteligentne wskaźniki

Stosowanie inteligentnych wskaźników (np. `boost::shared_ptr<T>` lub `std::shared_ptr<T>`) to kolejny powszechny w C++ sposób przekazywania współdzielonej własności obiektów.
Takie wskaźniki są obsługiwane automatycznie, jeśli zadeklarujemy typ przechowujący (holder type) jako parametr szablonu `class_<>` podczas rejestrowania klasy w Boost.Python:

```cpp
#include <string>
#include <boost/shared_ptr.hpp>
#include <boost/python.hpp>

using namespace boost;
using namespace std;
using namespace boost::python;

struct A {
    static shared_ptr<A> create () { return shared_ptr<A>(new A); }
    std::string   hello  () { return "Just nod if you can hear me!"; }
};

BOOST_PYTHON_MODULE(shared_ptr)
{
    class_<A, shared_ptr<A> >("A", init<>())
        .def("create", &A::create)
        .staticmethod("create")
        .def("hello", &A::hello)
    ;
}
```

## Operatory Pythona i metody specjalne

C++ posiada bogaty zestaw dobrze zdefiniowanych operatorów i umożliwia ich przeciążanie dla typów użytkownika.
Boost.Python wykorzystuje ten mechanizm, pozwalając w prosty i deklaratywny sposób mapować klasy C++ z przeciążonymi operatorami na odpowiadające im operatory Pythona.

Rozważmy klasę pozycji w pliku `FilePos` oraz zestaw operatorów działających na jej instancjach:

```cpp
class FilePos { /*...*/ };

FilePos     operator+(FilePos, int);
FilePos     operator+(int, FilePos);
int         operator-(FilePos, FilePos);
FilePos     operator-(FilePos, int);
FilePos&    operator+=(FilePos&, int);
bool        operator<(FilePos, FilePos);
```

Klasę wraz z jej operatorami można odwzorować w Pythonie bardzo czytelnie i intuicyjnie.
Poniższy zapis niemal bezpośrednio odzwierciedla sygnatury operatorów C++ — wystarczy pamiętać, że symbol `self` oznacza obiekt klasy `FilePos`:

```cpp
class_<FilePos>("FilePos")
    .def(self + int())          // __add__
    .def(int() + self)          // __radd__
    .def(self - self)           // __sub__
    .def(self - int())          // __sub__
    .def(self += int())         // __iadd__
    .def(self -= int())	        // __isub__
    .def(self < self);          // __lt__
```

Python udostępnia również zestaw metod specjalnych (dunder methods), takich jak np. `str()`.
Boost.Python wspiera standardowe nazwy metod specjalnych obsługiwanych przez klasy Pythona i udostępnia spójny zestaw pomocniczych wyrażeń do opakowywania odpowiadających im funkcji oraz operatorów C++.

Przykład:

```cpp
class Rational
{ public: operator double() const; };

Rational pow(Rational, Rational);
Rational abs(Rational);
ostream& operator<<(ostream&, Rational);

BOOST_PYTHON_MODULE(hello) {
	class_<Rational>("Rational")
    	.def(float_(self))                  // __float__
    	.def(pow(self, other<Rational>))    // __pow__
    	.def(abs(self))                     // __abs__
    	.def(str(self))                     // __str__
    ;
}
```

Warto zwrócić uwagę, że definicja `.def(str(self))` wykorzystuje przeciążony w C++ operator strumieniowy `operator<<`.

## Zwracanie list i krotek

Boost.Python udostępnia klasy opakowujące natywne kontenery Pythona, w tym krotki (`tuple`) oraz listy (`list`):

- `boost::python::tuple` można utworzyć za pomocą konstruktora lub wywołania funkcji pomocniczej `make_tuple(args)`. Obsługiwane są wszystkie wbudowane typy C++, a także nasze własne klasy, o ile wcześniej zarejestrowano dla nich wrapper `class_<>`.
- `boost::python::list` może przechowywać elementy dowolnego typu (podobnie jak krotki obsługuje konwersję typów C++).
	- listy wspierają standardowe operacje znane z Pythona: `append`, `extend`, `insert`, `count`, `pop`, `remove`.

Więcej szczegółów na temat pracy z listami i krotkami znajduje się w oficjalnej dokumentacji: [Reference Manual for Boost.Python](https://www.boost.org/doc/libs/1_66_0/libs/python/doc/html/reference/).

## Budowanie projektu

Standardowym narzędziem wykorzystywanym historycznie do budowania modułów Boost.Python (oraz samej biblioteki Boost) jest `bjam` (`b2`).
Poniżej znajdują się przykłady konfiguracji alternatywnych systemów budowania — SCons oraz CMake — do kompilacji przykładowego modułu rozszerzenia.

### SCons

```python
BOOST_VERSION = 'boost.cvs'
BOOST = '/usr/local/src/' + BOOST_VERSION
BOOSTLIBPATH = BOOST+'/stage/lib'
env = Environment (LIBPATH=['./',BOOSTLIBPATH], CPPPATH=[BOOST, '/usr/include/python'],
                   RPATH=['./',BOOSTLIBPATH])
env.SharedLibrary (target='uvector', source='uvector.cc', SHLIBPREFIX='', LIBS=[BOOST_PYTHON_LIB])
```

### CMake

```cmake
cmake_minimum_required(VERSION 3.5)

SET(ENV{BOOST_ROOT} "/path/to/my/boost") # ustaw, jeśli find_package nie znajduje Boosta w domyślnych ścieżkach

find_package(Boost COMPONENTS python REQUIRED)
find_package(PythonLibs 3.6 REQUIRED)

INCLUDE_DIRECTORIES(${Boost_INCLUDE_DIRS})
INCLUDE_DIRECTORIES(${PYTHON_INCLUDE_DIRS})

ADD_LIBRARY(MyLibrary SHARED MyLibraryInterface.cpp)
TARGET_LINK_LIBRARIES(MyLibrary "Boost::python" ${PYTHON_LIBRARIES})
```

##### Źródła i dalsza lektura

1. [Reference Manual for Boost.Python](https://www.boost.org/doc/libs/1_66_0/libs/python/doc/html/reference/)
2. [*Building Hybrid Systems with Boost.Python*](https://www.boost.org/doc/libs/1_66_0/libs/python/doc/html/article.html)
3. [Boost.Python sources](https://github.com/boostorg/python)
4. [Detailed CallPolicy](https://www.boost.org/doc/libs/1_66_0/libs/python/doc/html/reference/function_invocation_and_creation/models_of_callpolicies.html)
