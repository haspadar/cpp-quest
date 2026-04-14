# 06. Один класс — один файл

В прошлых уроках мы создали `Weapon`, `Armor`, `Player`, `Monster`, `Potion`. Пять классов. Все в одном файле. Файл перевалил за 200 строк. Найти нужный метод — листать и листать.

## Боль одного файла

Открой свой `main.cpp` из прошлого урока. Попробуй быстро ответить:
- На какой строке начинается класс `Armor`?
- Где метод `Player::takeDamage`?
- Где конструктор `Monster`?

Если пришлось скроллить — проблема уже есть. А будет хуже: добавим инвентарь, магазин, квесты — и файл станет на 500, 1000 строк.

## Каждый класс — отдельный файл

В C++ класс разделяют на два файла:
- **`.h`** (header) — объявление: какие поля и методы есть
- **`.cpp`** (source) — реализация: как методы работают

### Weapon.h — объявление

```cpp
#pragma once
#include <string>
#include <iostream>

class Weapon {
private:
    std::string name;
    int damage;

public:
    Weapon();
    Weapon(std::string n, int d);

    int strike() const;

    friend std::ostream& operator<<(std::ostream& out, const Weapon& w);
};
```

`#pragma once` — защита от повторного подключения. Если два файла подключают `Weapon.h` — он прочитается только один раз.

### Weapon.cpp — реализация

```cpp
#include "Weapon.h"

Weapon::Weapon() : name("Fist"), damage(5) {}

Weapon::Weapon(std::string n, int d) : name(n), damage(d) {}

int Weapon::strike() const {
    return damage;
}

std::ostream& operator<<(std::ostream& out, const Weapon& w) {
    out << w.name << " (урон: " << w.damage << ")";
    return out;
}
```

`Weapon::` перед каждым методом — говорит компилятору: «это метод класса Weapon, а не просто функция».

`#include "Weapon.h"` в кавычках — подключаем свой файл. `#include <iostream>` в угловых скобках — стандартную библиотеку.

### main.cpp — использование

```cpp
#include <iostream>
#include <Windows.h>
#include "Weapon.h"

int main() {
    SetConsoleCP(1251);
    SetConsoleOutputCP(1251);

    Weapon sword("Sword", 25);
    std::cout << sword << "\n";

    return 0;
}
```

`main.cpp` не знает, как устроен `Weapon` внутри. Он видит только `.h` — интерфейс. Реализация спрятана в `.cpp`.

### Компиляция

Теперь компилируем все `.cpp` файлы вместе:

```bash
g++ -std=c++17 -Wall -Wextra -o main main.cpp Weapon.cpp Player.cpp Armor.cpp
```

## Структура проекта

После разделения проект выглядит так:

```
cpp-quest/
├── Weapon.h
├── Weapon.cpp
├── Armor.h
├── Armor.cpp
├── Player.h
├── Player.cpp
├── Monster.h
├── Monster.cpp
└── main.cpp
```

Нужен `Weapon`? Открой `Weapon.h` — там интерфейс. Хочешь увидеть реализацию — `Weapon.cpp`. Не нужно листать 500 строк.

## Массивы объектов

В игре бывает не один монстр, а десять. Не одно зелье, а пять. Для этого — массивы:

```cpp
Monster horde[3] = {
    Monster("Goblin", 40, 10),
    Monster("Orc", 80, 20),
    Monster("Troll", 120, 15)
};
```

Чтобы массив работал, классу нужен **конструктор по умолчанию** (или заполнять массив сразу, как выше).

Обход массива:

```cpp
for (int i = 0; i < 3; i++) {
    std::cout << horde[i] << "\n";
}
```

## Отряд как объект

Массив персонажей — это **отряд**. А отряд — это объект:

```cpp
class Party {
private:
    Player members[5];
    int count = 0;

public:
    void add(Player p) {
        if (count < 5) {
            members[count] = p;
            count++;
        }
    }

    void printStatus() const {
        std::cout << "Отряд (" << count << " чел.):\n";
        for (int i = 0; i < count; i++) {
            std::cout << "  " << members[i] << "\n";
        }
    }

    bool isAlive() const {
        for (int i = 0; i < count; i++) {
            if (members[i].isAlive()) return true;
        }
        return false;
    }
};
```

Снаружи `Party` выглядит как один объект: `party.isAlive()`, `party.printStatus()`. Внутри — массив. Это композиция из урока 04, но с массивом.

## Полный пример

Файлов будет много, покажем ключевые части.

**Monster.h:**

```cpp
#pragma once
#include <string>
#include <iostream>

class Monster {
private:
    std::string name;
    int hp;
    int damage;

public:
    Monster();
    Monster(std::string n, int h, int d);

    void takeDamage(int dmg);
    bool isAlive() const;
    std::string getName() const;
    int getDamage() const;

    friend std::ostream& operator<<(std::ostream& out, const Monster& m);
};
```

**Monster.cpp:**

```cpp
#include "Monster.h"

Monster::Monster() : name("Unknown"), hp(1), damage(1) {}

Monster::Monster(std::string n, int h, int d) : name(n), hp(h), damage(d) {}

void Monster::takeDamage(int dmg) {
    hp -= dmg;
    if (hp < 0) hp = 0;
}

bool Monster::isAlive() const { return hp > 0; }
std::string Monster::getName() const { return name; }
int Monster::getDamage() const { return damage; }

std::ostream& operator<<(std::ostream& out, const Monster& m) {
    out << m.name << ": " << m.hp << " HP, урон: " << m.damage;
    return out;
}
```

**main.cpp:**

```cpp
#include <iostream>
#include <Windows.h>
#include "Player.h"
#include "Monster.h"

int main() {
    SetConsoleCP(1251);
    SetConsoleOutputCP(1251);

    Player knight("Knight", 100, Weapon("Sword", 30), Armor("Iron Plate", 10));

    Monster horde[3] = {
        Monster("Goblin", 40, 10),
        Monster("Orc", 80, 20),
        Monster("Troll", 120, 15)
    };

    std::cout << knight << "\n\n";

    for (int i = 0; i < 3; i++) {
        std::cout << "=== Бой с " << horde[i].getName() << " ===\n";

        while (knight.isAlive() && horde[i].isAlive()) {
            knight.attack(horde[i]);
            if (horde[i].isAlive()) {
                knight.takeDamage(horde[i].getDamage());
            }
        }

        std::cout << knight << "\n\n";

        if (!knight.isAlive()) {
            std::cout << "Герой пал...\n";
            break;
        }
    }

    if (knight.isAlive()) {
        std::cout << "Все монстры побеждены!\n";
    }

    return 0;
}
```

## Шпаргалка

- Один класс = один `.h` + один `.cpp`
- `.h` — объявление (что есть): поля, прототипы методов
- `.cpp` — реализация (как работает): тела методов с `ClassName::`
- `#pragma once` — защита от повторного подключения
- `#include "MyClass.h"` — свои файлы в кавычках
- `#include <iostream>` — стандартные в угловых скобках
- Компиляция: `g++ -o main main.cpp Weapon.cpp Player.cpp ...`
- Массив объектов: `Monster horde[3] = { ... };`
- Коллекция — тоже объект: `Party` хранит массив `Player`

```
MyClass.h → что класс умеет (интерфейс)
MyClass.cpp → как класс это делает (реализация)
main.cpp → использование
```

## Напоследок

1. В Java один класс = один файл — это правило языка, а не конвенция. Файл `Player.java` **обязан** содержать класс `Player`. В C++ это договорённость, но настолько устоявшаяся, что нарушать её не принято.

2. Разделение на `.h` и `.cpp` — уникальная особенность C и C++. В Python, Java, C#, PHP, JavaScript — один файл на класс, без разделения на «объявление» и «реализацию». Это наследие эпохи, когда компилятору нужно было заранее знать, что существует, прежде чем использовать.

3. `#pragma once` — нестандартное, но поддерживаемое всеми компиляторами расширение. Старый способ — `#ifndef / #define / #endif` — делает то же самое, но в три строки. В университетских примерах часто встречается старый способ — теперь ты знаешь, что это одно и то же.

## Практика

### 6.1. Разделение на файлы

Возьми классы из прошлых уроков: `Weapon`, `Armor`, `Player`, `Monster`, `Potion`. Раздели каждый на `.h` и `.cpp`. Убедись, что проект компилируется.

### 6.2. Орда монстров

Создай массив из 5 монстров с разными характеристиками. Герой сражается с ними по очереди. После каждого боя — вывод состояния героя. Между боями герой может использовать зелье (если HP ниже 50).

```text
=== Бой 1: Knight vs Goblin ===
Knight победил! Knight: 85/100 HP
=== Бой 2: Knight vs Orc ===
Knight использует зелье! HP: 65 -> 95
Knight победил! Knight: 55/100 HP
...
```

### 6.3. Отряд

Создай класс `Party` в отдельных файлах (`Party.h`, `Party.cpp`). Отряд хранит до 4 игроков.

Методы:
- `add(Player p)` — добавить в отряд
- `isAlive()` — жив ли кто-то
- `printStatus()` — статус всех членов
- `operator<<` — вывод отряда

Проведи бой: отряд из 3 героев vs массив из 5 монстров. Каждый ход каждый живой герой атакует текущего монстра. Монстр бьёт случайного живого героя.

```cpp
Party heroes;
heroes.add(Player("Knight", 100, Weapon("Sword", 30), Armor("Iron Plate", 10)));
heroes.add(Player("Mage", 60, Weapon("Staff", 40), Armor("Robe", 2)));
heroes.add(Player("Archer", 80, Weapon("Bow", 25), Armor("Leather", 5)));

std::cout << heroes << "\n";
```
