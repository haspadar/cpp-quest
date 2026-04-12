# 05. Разделение на файлы и массивы объектов

До сих пор весь код жил в одном файле. Для маленьких примеров это нормально, но когда классов становится несколько — файл превращается в кашу. Пора научиться разделять код.

## Зачем разделять

Представь: у тебя класс `Player` (40 строк), класс `Weapon` (30 строк), класс `Potion` (25 строк) и `main` (50 строк). Итого 145 строк в одном файле. И это только начало — в реальном проекте будут десятки классов.

Разделение даёт:
- каждый класс в своём файле — легко найти
- несколько человек могут работать с разными файлами одновременно
- изменил один класс — перекомпилировался только его файл

## Структура: .h и .cpp

Каждый класс разбивается на два файла:

**Заголовочный файл (.h)** — объявление класса: какие поля, какие методы. Это "контракт" — что умеет класс.

**Файл реализации (.cpp)** — код методов. Это "начинка" — как именно работает.

### Player.h — объявление

```cpp
#pragma once

#include <string>
#include <iostream>

class Player {
private:
    std::string name;
    int hp;
    int maxHp;

public:
    Player();
    Player(std::string name, int hp);
    ~Player();

    std::string getName() const;
    int getHp() const;

    void takeDamage(int dmg);
    void heal(int amount);
    bool isAlive() const;
    void printStatus() const;
};
```

`#pragma once` — говорит компилятору: "подключай этот файл только один раз". Без этого, если два файла подключат `Player.h`, компилятор увидит два объявления класса и выдаст ошибку.

Обрати внимание: методы только объявлены (прототипы), но не реализованы. После каждого прототипа стоит точка с запятой.

### Player.cpp — реализация

```cpp
#include "Player.h"

Player::Player() : name("Unknown"), hp(100), maxHp(100) {}

Player::Player(std::string name, int hp)
    : name(name), hp(hp), maxHp(hp) {}

Player::~Player() {}

std::string Player::getName() const { return name; }
int Player::getHp() const { return hp; }

void Player::takeDamage(int dmg) {
    hp -= dmg;
    if (hp < 0) hp = 0;
}

void Player::heal(int amount) {
    hp += amount;
    if (hp > maxHp) hp = maxHp;
}

bool Player::isAlive() const {
    return hp > 0;
}

void Player::printStatus() const {
    std::cout << name << ": " << hp << "/" << maxHp << " HP\n";
}
```

`Player::` перед каждым методом — говорит: "это метод класса `Player`, а не просто функция". Без этого компилятор не поймёт, к какому классу относится метод.

`#include "Player.h"` — в кавычках, потому что это наш файл, а не системная библиотека. Системные пишутся в угловых скобках: `<iostream>`.

### main.cpp — использование

```cpp
#include <iostream>
#include <Windows.h>
#include "Player.h"

int main() {
    SetConsoleCP(1251);
    SetConsoleOutputCP(1251);

    Player knight("Knight", 100);
    knight.printStatus();

    knight.takeDamage(30);
    knight.printStatus();

    return 0;
}
```

В CLion всё соберётся автоматически — просто добавь файлы в проект. Если в проекте используется CMake, CLion подхватит новые `.cpp` файлы сам (при условии, что они в правильной директории и добавлены в `CMakeLists.txt`).

## Массивы объектов

Пока мы создавали объекты по одному: `Player knight`, `Player mage`, `Player rogue`. Три персонажа — три переменные. А если в отряде 10 героев? Или 100? Создавать отдельную переменную для каждого — то же самое, что в уроке 02 мы делали с `name1`, `age1`, `name2`, `age2`.

Решение то же — массив, только теперь массив объектов:

```cpp
Player party[3];
```

Это создаёт 3 объекта `Player`. Для каждого вызывается **конструктор по умолчанию**. Если его нет — ошибка компиляции.

Вот почему конструктор по умолчанию важен: без него нельзя создать массив.

### Заполнение массива

```cpp
Player party[3];

party[0] = Player("Knight", 100);
party[1] = Player("Mage", 70);
party[2] = Player("Rogue", 80);
```

Или через цикл с вводом:

```cpp
const int SIZE = 3;
Player party[SIZE];

for (int i = 0; i < SIZE; i++) {
    std::string name;
    int hp;
    std::cout << "Имя: ";
    std::cin >> name;
    std::cout << "HP: ";
    std::cin >> hp;
    party[i] = Player(name, hp);
}
```

### Работа с массивом

Вывод всех:

```cpp
for (int i = 0; i < SIZE; i++) {
    party[i].printStatus();
}
```

Поиск самого сильного:

```cpp
int maxIndex = 0;
for (int i = 1; i < SIZE; i++) {
    if (party[i].getHp() > party[maxIndex].getHp()) {
        maxIndex = i;
    }
}
std::cout << "Самый живучий: " << party[maxIndex].getName() << "\n";
```

Вот здесь геттер `getHp()` действительно нужен — мы сравниваем объекты между собой, и без доступа к значению HP это не сделать.

## Полный пример

Файл `Player.h`:

```cpp
#pragma once

#include <string>
#include <iostream>

class Player {
private:
    std::string name = "Unknown";
    int hp = 100;
    int maxHp = 100;

public:
    Player();
    Player(std::string name, int hp);

    std::string getName() const;
    int getHp() const;

    void takeDamage(int dmg);
    bool isAlive() const;
    void printStatus() const;
};
```

Файл `Player.cpp`:

```cpp
#include "Player.h"

Player::Player() {}

Player::Player(std::string name, int hp)
    : name(name), hp(hp), maxHp(hp) {}

std::string Player::getName() const { return name; }
int Player::getHp() const { return hp; }

void Player::takeDamage(int dmg) {
    hp -= dmg;
    if (hp < 0) hp = 0;
}

bool Player::isAlive() const { return hp > 0; }

void Player::printStatus() const {
    std::cout << name << ": " << hp << "/" << maxHp << " HP\n";
}
```

Файл `main.cpp`:

```cpp
#include <iostream>
#include <Windows.h>
#include "Player.h"

int main() {
    SetConsoleCP(1251);
    SetConsoleOutputCP(1251);

    const int SIZE = 3;
    Player party[SIZE];

    party[0] = Player("Knight", 100);
    party[1] = Player("Mage", 70);
    party[2] = Player("Rogue", 80);

    std::cout << "Отряд:\n";
    for (int i = 0; i < SIZE; i++) {
        party[i].printStatus();
    }

    std::cout << "\nВсе получают 40 урона:\n";
    for (int i = 0; i < SIZE; i++) {
        party[i].takeDamage(40);
        party[i].printStatus();
    }

    std::cout << "\nВыжившие:\n";
    for (int i = 0; i < SIZE; i++) {
        if (party[i].isAlive()) {
            std::cout << party[i].getName() << " ещё жив!\n";
        }
    }

    return 0;
}
```

```text
Отряд:
Knight: 100/100 HP
Mage: 70/70 HP
Rogue: 80/80 HP

Все получают 40 урона:
Knight: 60/100 HP
Mage: 30/70 HP
Rogue: 40/80 HP

Выжившие:
Knight ещё жив!
Mage ещё жив!
Rogue ещё жив!
```

## Шпаргалка

**Файлы:**
- `.h` — объявление класса (поля + прототипы методов)
- `.cpp` — реализация методов через `ClassName::method()`
- `#pragma once` — защита от повторного подключения
- `#include "MyClass.h"` — свои файлы в кавычках
- `#include <iostream>` — системные в угловых скобках

**Массивы объектов:**
- `Player party[3]` — массив из 3 объектов
- Нужен конструктор по умолчанию
- Работа через индекс: `party[i].method()`

## Практика

### 5.1. Вынеси класс в файлы

Возьми класс `Weapon` из урока 03. Раздели на `Weapon.h` и `Weapon.cpp`. В `main.cpp` создай оружие и выведи информацию.

### 5.2. Арсенал

Создай массив из 5 единиц оружия. Заполни его — можно вручную или вводом с клавиатуры. Выведи всё оружие и найди самое мощное (с максимальным уроном).

```text
Арсенал:
Sword (урон: 25)
Bow (урон: 15)
Axe (урон: 30)
Dagger (урон: 10)
Staff (урон: 20)

Самое мощное: Axe (урон: 30)
```

### 5.3. Отряд героев

Создай программу, которая:
1. Спрашивает количество героев (N)
2. Для каждого запрашивает имя и HP
3. Выводит весь отряд
4. Наносит каждому случайный урон от 20 до 60
5. Выводит выживших

Класс `Player` — в отдельных файлах (`Player.h` + `Player.cpp`).

Подсказка: для случайного числа от 20 до 60:
```cpp
#include <cstdlib>
#include <ctime>

srand(time(0));              // в начале main, один раз
int dmg = rand() % 41 + 20; // от 20 до 60
```
