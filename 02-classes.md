# 02. Классы и объекты: создаём первого персонажа

В прошлом уроке мы перешли с C на C++: `cout`, `cin`, `std::string`. Теперь — первое знакомство с ООП.

## Зачем вообще классы

Допустим, мы пишем программу про животных. У каждого есть кличка и возраст.

Можно хранить всё в отдельных переменных:

```cpp
std::string name1 = "Bobik";
int age1 = 3;

std::string name2 = "Murzik";
int age2 = 5;
```

Два животных — уже 4 переменных. Десять — 20. И ничего не связано: легко перепутать `age1` с `age2`, передать не те данные в функцию.

Структура решает эту проблему — объединяет данные в одно целое:

```cpp
struct Animal {
    std::string name;
    int age;
};
```

Теперь:

```cpp
Animal dog;
dog.name = "Bobik";
dog.age = 3;
```

Один объект — все данные вместе, ничего не перепутаешь.

## Проблема со структурами

Всё работает, но есть подвох. Никто не мешает написать:

```cpp
dog.age = -10;
```

Компилятор не ругается. Программа не падает. Но собака с возрастом −10 лет — это баг.

Структура не защищает свои данные. Любой код может записать в поле что угодно.

## Класс = структура + защита

Класс — это та же структура, но с контролем доступа.

Два ключевых слова:
- `private` — поля и методы, доступные только изнутри класса
- `public` — поля и методы, доступные снаружи

```cpp
class Animal {
private:
    std::string name;
    int age;

public:
    // методы будут здесь
};
```

Теперь `age` нельзя изменить напрямую:

```cpp
Animal dog;
dog.age = -10; // ошибка компиляции
```

Данные спрятаны. Чтобы работать с животным, нужны методы.

## Методы

Метод — это функция внутри класса. Она имеет доступ к private-полям.

```cpp
class Animal {
private:
    std::string name;
    int age;

public:
    void print() {
        std::cout << name << ", " << age << " лет\n";
    }

    void birthday() {
        age++;
    }

    void rename(std::string newName) {
        name = newName;
    }
};
```

`print`, `birthday`, `rename` — методы. Они видят `name` и `age`, хотя те private. Код снаружи класса — не видит.

Обрати внимание на названия:
- `birthday`, а не `setAge` — мы не «устанавливаем возраст», у животного наступает день рождения
- `rename`, а не `setName` — мы переименовываем, а не «присваиваем строку в поле»

Методы описывают действия из жизни, а не технические операции над переменными.

## Конструктор

Поля private — снаружи нельзя написать `dog.name = "Bobik"`. Нужен способ задать начальные значения.

**Конструктор** — специальный метод, который вызывается при создании объекта. Его имя совпадает с именем класса:

```cpp
class Animal {
private:
    std::string name;
    int age;

public:
    Animal(std::string n, int a) {
        name = n;
        age = a;
    }

    void print() {
        std::cout << name << ", " << age << " лет\n";
    }

    void birthday() {
        age++;
    }
};
```

Теперь:

```cpp
Animal dog("Bobik", 3);
dog.print(); // Bobik, 3 лет
```

Одна строка — и объект готов.

## Полный пример

```cpp
#include <iostream>
#include <string>
#include <Windows.h>

class Animal {
private:
    std::string name;
    int age;

public:
    Animal(std::string n, int a) {
        name = n;
        age = a;
    }

    void print() {
        std::cout << name << ", " << age << " лет\n";
    }

    void birthday() {
        age++;
    }

    void rename(std::string newName) {
        name = newName;
    }
};

int main() {
    SetConsoleCP(1251);
    SetConsoleOutputCP(1251);

    Animal dog("Bobik", 3);
    Animal cat("Murzik", 5);

    dog.print();
    cat.print();

    std::cout << "\nУ Bobik день рождения!\n";
    dog.birthday();
    dog.print();

    std::cout << "\nBobik теперь Rex!\n";
    dog.rename("Rex");
    dog.print();

    return 0;
}
```

```text
Bobik, 3 лет
Murzik, 5 лет

У Bobik день рождения!
Bobik, 4 лет

Bobik теперь Rex!
Rex, 4 лет
```

## Шпаргалка

- `struct` — все поля открыты по умолчанию, нет защиты данных
- `class` — все поля закрыты по умолчанию (`private`)
- `private:` — доступно только внутри класса
- `public:` — доступно снаружи
- Метод — функция внутри класса, имеет доступ к private-полям
- Конструктор — метод с именем класса, вызывается при создании объекта
- Методы называем как действия: `birthday` а не `setAge`, `rename` а не `setName`

```cpp
class Animal {
private:
    std::string name;
    int age;

public:
    Animal(std::string n, int a) { name = n; age = a; }
    void print() { std::cout << name << ", " << age << " лет\n"; }
    void birthday() { age++; }
    void rename(std::string newName) { name = newName; }
};
```

## Практика

### 2.1. Персонаж получает урон

Возьми класс `Player` из примера выше. Сейчас он умеет только показывать себя.

Добавь метод, который уменьшает здоровье на переданное число. Придумай ему название сам — как бы это звучало в игре?

Требование: здоровье не должно уходить ниже 0.

Проверь:

```cpp
Player mage("Mage", 80);
mage.printStatus();
// нанеси 30 урона
mage.printStatus();
// нанеси ещё 100 урона
mage.printStatus();
```

Ожидаемый вывод:

```text
Mage: 80 HP
Mage: 50 HP
Mage: 0 HP
```

### 2.2. Лечение и проверка

Добавь ещё два метода:
- лечение — увеличивает здоровье, но не выше 100
- проверка — возвращает `true`, если персонаж жив (здоровье больше 0)

Названия придумай сам.

Проверь:

```cpp
Player warrior("Warrior", 100);
// нанеси 90 урона
warrior.printStatus();
// вылечи на 50
warrior.printStatus();
std::cout << "Alive: " << warrior.isAlive() << "\n"; // или твоё название
```

Ожидаемый вывод:

```text
Warrior: 10 HP
Warrior: 60 HP
Alive: 1
```

### 2.3. Бой двух персонажей

Создай двух персонажей:
- `Knight` — 100 HP
- `Rogue` — 80 HP

Knight наносит 25 урона за удар, Rogue — 35.

Напиши цикл: они бьют друг друга по очереди, пока один не погибнет. После каждого удара — вывод состояния обоих.

Подумай: силу удара пока можно задать обычной переменной. Но что, если сделать её полем класса и добавить метод, который сам наносит урон другому игроку? Попробуй оба варианта.
