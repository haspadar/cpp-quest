# 03. Объекты рождаются готовыми

В прошлом уроке мы создали класс `Player` с конструктором — передали имя и здоровье при создании. Теперь разберём конструкторы подробнее и познакомимся с деструкторами.

## Полуготовый объект — это баг

Представь: ты создаёшь оружие и забываешь задать урон.

```cpp
Weapon sword;
sword.print(); // Sword (урон: 0)... или мусор из памяти?
```

Оружие без урона — это не оружие. Персонаж без имени — это не персонаж. Зелье без силы — это не зелье.

Если объект можно создать в «недоделанном» состоянии — рано или поздно кто-то так и сделает. А потом полдня будет искать баг.

Правило: **объект должен быть готов к работе с момента рождения**. Конструктор — это гарантия.

## Конструктор с параметрами

В уроке 02 мы уже писали конструктор:

```cpp
class Player {
private:
    std::string name;
    int hp;

public:
    Player(std::string n, int h) {
        name = n;
        hp = h;
    }
};
```

Теперь `Player hero("Knight", 100)` — и герой сразу готов. Нельзя создать игрока без имени, компилятор не даст:

```cpp
Player hero; // ошибка компиляции!
```

И это хорошо. Компилятор ловит ошибку за нас.

## Конструктор по умолчанию

Но иногда нужны разумные начальные значения. Оружие по умолчанию — кулаки:

```cpp
class Weapon {
private:
    std::string name;
    int damage;

public:
    Weapon() {
        name = "Fist";
        damage = 5;
    }

    Weapon(std::string n, int d) {
        name = n;
        damage = d;
    }

    void print() {
        std::cout << name << " (урон: " << damage << ")\n";
    }
};
```

Теперь можно и так, и так:

```cpp
Weapon fist;                    // Fist, урон 5
Weapon sword("Sword", 25);     // Sword, урон 25
```

Оба варианта — готовый объект. `Fist` с уроном 5 — это полноценное оружие, не «пустышка».

Важно: если у класса есть конструктор с параметрами и нет конструктора без параметров — компилятор не создаст его автоматически. Либо пиши оба, либо только с параметрами.

## Инициализация полей в объявлении

Вместо того чтобы присваивать значения в конструкторе по умолчанию, можно задать их прямо при объявлении полей:

```cpp
class Weapon {
private:
    std::string name = "Fist";
    int damage = 5;

public:
    Weapon() {}  // тело пустое — поля уже инициализированы

    Weapon(std::string n, int d) {
        name = n;
        damage = d;
    }
};
```

Если конструктор не трогает поле — оно получит значение из объявления. Если трогает — перезапишет.

## Список инициализации

Посмотри на конструктор:

```cpp
Weapon(std::string n, int d) {
    name = n;
    damage = d;
}
```

Здесь происходит два шага: сначала `name` создаётся пустой строкой, потом ей присваивается `n`. Создали пустую — тут же выбросили и записали другую. Лишняя работа.

**Список инициализации** делает это за один шаг. Пишется через двоеточие после скобок:

```cpp
Weapon(std::string n, int d) : name(n), damage(d) {
    // тело пустое — всё уже задано
}
```

Поле сразу создаётся с нужным значением. Для `int` разница незаметна. Для строк и сложных объектов — ощутима.

Правило: если можешь использовать список инициализации — используй.

## Деструктор

Мы научились создавать объекты. Но что происходит, когда объект больше не нужен?

```cpp
void battle() {
    Weapon sword("Sword", 25);  // конструктор
    // ... бой ...
}  // здесь sword уничтожается — вызывается деструктор
```

**Деструктор** — метод, который вызывается автоматически при уничтожении объекта. Пишется как конструктор, но с тильдой `~`:

```cpp
class Weapon {
private:
    std::string name;
    int damage;

public:
    Weapon(std::string n, int d) : name(n), damage(d) {
        std::cout << name << " создан\n";
    }

    ~Weapon() {
        std::cout << name << " уничтожен\n";
    }
};
```

Деструктор не принимает параметров и ничего не возвращает. У класса может быть только один.

Зачем он нужен? Сейчас деструктор просто выводит текст. Но в реальных программах объект может открыть файл, выделить память, подключиться к сети. Деструктор — место, где всё это освобождается. Если класс не выделяет ресурсов — деструктор можно не писать, компилятор создаст пустой.

## Область видимости и порядок уничтожения

Объект живёт, пока жива его область видимости — блок `{}`:

```cpp
int main() {
    Weapon bow("Bow", 15);          // 1. создан Bow
    {
        Weapon dagger("Dagger", 10); // 2. создан Dagger
    }                                // 3. уничтожен Dagger (блок закончился)
    std::cout << "Bow ещё жив\n";   // 4. Bow жив
    return 0;
}                                    // 5. уничтожен Bow
```

Объекты уничтожаются в **обратном** порядке создания. Как стопка тарелок: последнюю поставил — первую снял.

## Полный пример

```cpp
#include <iostream>
#include <string>
#include <Windows.h>

class Weapon {
private:
    std::string name = "Fist";
    int damage = 5;

public:
    Weapon() {
        std::cout << "Создано оружие по умолчанию: " << name << "\n";
    }

    Weapon(std::string n, int d) : name(n), damage(d) {
        std::cout << "Создано оружие: " << name << "\n";
    }

    ~Weapon() {
        std::cout << "Оружие " << name << " уничтожено\n";
    }

    void print() {
        std::cout << name << " (урон: " << damage << ")\n";
    }

    int getDamage() const { return damage; }
    std::string getName() const { return name; }
};

class Player {
private:
    std::string name;
    int hp;
    int damage;

public:
    Player(std::string n, int h, int d) : name(n), hp(h), damage(d) {
        std::cout << name << " вступает в бой!\n";
    }

    ~Player() {
        std::cout << name << " покидает арену\n";
    }

    void attack(Player& target) {
        std::cout << name << " бьёт " << target.name << "\n";
        target.takeDamage(damage);
    }

    void takeDamage(int dmg) {
        hp -= dmg;
        if (hp < 0) hp = 0;
    }

    bool isAlive() const { return hp > 0; }

    void printStatus() const {
        std::cout << name << ": " << hp << " HP\n";
    }
};

int main() {
    SetConsoleCP(1251);
    SetConsoleOutputCP(1251);

    Weapon sword("Sword", 25);
    sword.print();

    Weapon fist;
    fist.print();

    std::cout << "\n";

    {
        Player knight("Knight", 100, 25);
        Player rogue("Rogue", 80, 35);

        while (knight.isAlive() && rogue.isAlive()) {
            knight.attack(rogue);
            if (rogue.isAlive()) {
                rogue.attack(knight);
            }
            knight.printStatus();
            rogue.printStatus();
            std::cout << "---\n";
        }
    }

    std::cout << "\nБой окончен, оружие ещё существует:\n";
    sword.print();

    return 0;
}
```

```text
Создано оружие: Sword
Sword (урон: 25)
Создано оружие по умолчанию: Fist
Fist (урон: 5)

Knight вступает в бой!
Rogue вступает в бой!
Knight бьёт Rogue
Rogue бьёт Knight
Knight: 65 HP
Rogue: 55 HP
---
Knight бьёт Rogue
Rogue бьёт Knight
Knight: 30 HP
Rogue: 30 HP
---
Knight бьёт Rogue
Rogue бьёт Knight
Knight: -5 HP
Rogue: 5 HP
---
Rogue покидает арену
Knight покидает арену

Бой окончен, оружие ещё существует:
Sword (урон: 25)
Оружие Fist уничтожено
Оружие Sword уничтожено
```

Обрати внимание: игроки создаются и умирают внутри блока `{}`. Оружие живёт дольше — оно в `main()`.

## Шпаргалка

- Объект должен быть готов с момента создания — конструктор это гарантирует
- Конструктор по умолчанию — без параметров: `Weapon() {}`
- Конструктор с параметрами: `Weapon(string n, int d)`
- Инициализация полей в объявлении: `int damage = 5;`
- Список инициализации: `Weapon(string n, int d) : name(n), damage(d) {}`
- Деструктор: `~Weapon()` — вызывается при уничтожении объекта
- Объекты уничтожаются в обратном порядке создания
- Область видимости `{}` определяет время жизни объекта

```cpp
class Weapon {
private:
    std::string name = "Fist";
    int damage = 5;
public:
    Weapon() {}
    Weapon(std::string n, int d) : name(n), damage(d) {}
    ~Weapon() {}
    void print() { std::cout << name << " (урон: " << damage << ")\n"; }
};
```

## Напоследок

1. Слово «конструктор» впервые появилось в языке Simula 67 — первом объектно-ориентированном языке, созданном в Норвегии для моделирования морских судов.

2. В Java и C# деструкторов нет — там за уборку отвечает сборщик мусора (garbage collector). Он сам решает, когда уничтожить объект. В C++ ты контролируешь это сам — объект умирает ровно тогда, когда заканчивается его область видимости. Это и сложнее, и мощнее.

3. В Python конструктор называется `__init__`, а деструктор — `__del__`. В PHP — `__construct` и `__destruct`. Идея одна и та же, синтаксис разный.

## Практика

### 3.1. Зелья

Создай класс `Potion` с полями: название и сила действия.

Два конструктора:
- по умолчанию: `"Small Heal"`, сила 20
- с параметрами: любые значения

Деструктор выводит: `"Зелье <name> использовано"`.

Проверь:

```cpp
Potion small;
small.print();

Potion big("Mega Heal", 100);
big.print();
```

```text
Зелье Small Heal создано
Small Heal (сила: 20)
Зелье Mega Heal создано
Mega Heal (сила: 100)
Зелье Mega Heal использовано
Зелье Small Heal использовано
```

### 3.2. Кто когда умрёт?

Что выведет эта программа? Сначала подумай, потом проверь.

```cpp
int main() {
    Weapon bow("Bow", 15);
    {
        Weapon dagger("Dagger", 10);
        dagger.print();
    }
    std::cout << "Между блоками\n";
    {
        Weapon staff("Staff", 20);
        staff.print();
    }
    bow.print();
    return 0;
}
```

### 3.3. Монстр со здоровьем

Создай класс `Monster` с полями: имя, здоровье, урон.

Конструктор принимает все три параметра — монстр без имени, здоровья и урона не бывает.

Добавь метод `attack(Player& target)` — монстр бьёт игрока. И метод `takeDamage(int dmg)` — монстр получает урон.

Проверь:

```cpp
Player hero("Knight", 100, 25);
Monster goblin("Goblin", 60, 15);

while (hero.isAlive() && goblin.isAlive()) {
    hero.attack(goblin);  // пока не работает — Player не умеет бить Monster
    goblin.attack(hero);
    hero.printStatus();
    goblin.printStatus();
    std::cout << "---\n";
}
```

Ты заметишь проблему: `Player::attack` принимает `Player&`, а не `Monster&`. А `Monster::attack` принимает `Player&`. Два класса знают друг о друге — это неудобно. Пока обойдись дублированием. В следующих уроках мы решим это красиво.
