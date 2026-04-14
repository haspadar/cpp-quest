# 10. Объекты наследуют

В прошлом уроке мы создали интерфейс `Fightable` — обещание, что объект умеет сражаться. И `Player`, и `Monster` его реализуют. Но у них много общего кода: оба хранят имя, HP, оба умеют `takeDamage`, `isAlive`. Дублирование.

## Копирование — это боль

Посмотри на `Player` и `Monster`:

```cpp
class Player : public Fightable {
private:
    std::string name;
    int hp;
    int maxHp;
public:
    void takeDamage(int dmg) override { hp -= dmg; if (hp < 0) hp = 0; }
    bool isAlive() const override { return hp > 0; }
    std::string getName() const override { return name; }
};

class Monster : public Fightable {
private:
    std::string name;
    int hp;
    int maxHp;
public:
    void takeDamage(int dmg) override { hp -= dmg; if (hp < 0) hp = 0; }
    bool isAlive() const override { return hp > 0; }
    std::string getName() const override { return name; }
};
```

Одинаковые поля. Одинаковые методы. Если найдёшь баг в `takeDamage` — чинить в двух местах. Добавишь третий тип (NPC, Boss) — в трёх.

## Общее — в базовый класс

**Наследование** позволяет вынести общий код в один класс:

```cpp
class Character : public Fightable {
protected:
    std::string name;
    int hp;
    int maxHp;

public:
    Character(std::string n, int h) : name(n), hp(h), maxHp(h) {}

    void takeDamage(int dmg) override {
        hp -= dmg;
        if (hp < 0) hp = 0;
    }

    bool isAlive() const override { return hp > 0; }
    std::string getName() const override { return name; }
};
```

`Character` — базовый класс. Он содержит всё общее: имя, HP, получение урона, проверку жизни.

`protected` — как `private`, но видно в производных классах.

## Производные классы

Теперь `Player` и `Monster` **наследуют** от `Character` и добавляют только своё:

```cpp
class Player : public Character {
private:
    Striker& weapon;
    int armor;

public:
    Player(std::string n, int h, Striker& w, int armor)
        : Character(n, h), weapon(w), armor(armor) {}

    void attack(Fightable& target) {
        int dmg = weapon.strike();
        std::cout << name << " атакует " << target.getName() << "!\n";
        target.takeDamage(dmg);
    }

    void takeDamage(int dmg) override {
        int actual = dmg - armor;
        if (actual < 0) actual = 0;
        Character::takeDamage(actual);
    }
};

class Monster : public Character {
private:
    int damage;

public:
    Monster(std::string n, int h, int d)
        : Character(n, h), damage(d) {}

    void attack(Fightable& target) {
        std::cout << name << " бьёт " << target.getName() << "!\n";
        target.takeDamage(damage);
    }
};
```

Разберём:

- `: public Character` — наследуем всё от `Character`
- `Character(n, h)` в списке инициализации — вызываем конструктор базового класса
- `Player` добавляет оружие и броню. `Monster` — силу атаки.
- `Player::takeDamage` переопределяет базовый метод: сначала вычитает броню, потом вызывает `Character::takeDamage`

## «Является» vs «Имеет»

Наследование — отношение **«является»**:
- Player **является** Character ✓
- Monster **является** Character ✓
- Weapon **является** Character ✗ — оружие не персонаж

Если отношение «является» звучит неестественно — используй композицию («имеет»):
- Player **имеет** Weapon ✓ (композиция, урок 04)
- Player **является** Character ✓ (наследование)

## Наследование — инструмент, а не цель

У тебя уже есть три способа расширять поведение:

1. **Композиция** (урок 04): Player имеет Weapon
2. **Декоратор** (урок 08): PoisonedWeapon оборачивает Weapon
3. **Наследование** (этот урок): Player наследует Character

Когда что использовать?

- **Композиция**: объект **имеет** другой объект. Player имеет Weapon.
- **Декоратор**: нужно **добавить поведение** без изменения оригинала. Яд на мече.
- **Наследование**: объекты **одного вида** с общим поведением. Player и Monster — оба Character.

Если можно решить композицией — решай композицией. Наследование создаёт жёсткую связь: изменил `Character` — изменились все наследники.

## Полный пример

```cpp
#include <iostream>
#include <string>
#include <cstdlib>
#include <ctime>
#include <Windows.h>

class Fightable {
public:
    virtual void takeDamage(int dmg) = 0;
    virtual bool isAlive() const = 0;
    virtual std::string getName() const = 0;
    virtual ~Fightable() {}
};

class Striker {
public:
    virtual int strike() const = 0;
    virtual ~Striker() {}
};

class Weapon : public Striker {
private:
    std::string name;
    int damage;
public:
    Weapon(std::string n, int d) : name(n), damage(d) {}
    int strike() const override { return damage; }
};

class Character : public Fightable {
protected:
    std::string name;
    int hp;
    int maxHp;
public:
    Character(std::string n, int h) : name(n), hp(h), maxHp(h) {}

    void takeDamage(int dmg) override {
        hp -= dmg;
        if (hp < 0) hp = 0;
    }

    bool isAlive() const override { return hp > 0; }
    std::string getName() const override { return name; }

    friend std::ostream& operator<<(std::ostream& out, const Character& c) {
        out << c.name << ": " << c.hp << "/" << c.maxHp << " HP";
        return out;
    }
};

class Player : public Character {
private:
    Striker& weapon;

public:
    Player(std::string n, int h, Striker& w)
        : Character(n, h), weapon(w) {}

    void attack(Fightable& target) {
        int dmg = weapon.strike();
        std::cout << name << " атакует " << target.getName()
                  << " на " << dmg << "!\n";
        target.takeDamage(dmg);
    }
};

class Goblin : public Character {
private:
    int damage;
public:
    Goblin(std::string n, int h, int d)
        : Character(n, h), damage(d) {}

    void attack(Fightable& target) {
        std::cout << name << " кусает " << target.getName() << "!\n";
        target.takeDamage(damage);
    }
};

class Dragon : public Character {
private:
    int breathDmg;
public:
    Dragon(std::string n, int h, int breath)
        : Character(n, h), breathDmg(breath) {}

    void attack(Fightable& target) {
        if (rand() % 100 < 70) {
            std::cout << name << " дышит огнём на " << target.getName() << "!\n";
            target.takeDamage(breathDmg);
        } else {
            std::cout << name << " промахивается!\n";
        }
    }
};

class Skeleton : public Character {
private:
    int damage;
    bool revived = false;

public:
    Skeleton(std::string n, int h, int d)
        : Character(n, h), damage(d) {}

    void takeDamage(int dmg) override {
        Character::takeDamage(dmg);
        if (hp <= 0 && !revived) {
            revived = true;
            hp = maxHp / 2;
            std::cout << name << " возрождается с " << hp << " HP!\n";
        }
    }

    void attack(Fightable& target) {
        std::cout << name << " бьёт костяным мечом!\n";
        target.takeDamage(damage);
    }
};

int main() {
    SetConsoleCP(1251);
    SetConsoleOutputCP(1251);
    srand(time(0));

    Weapon sword("Sword", 30);
    Player knight("Knight", 120, sword);

    Goblin goblin("Goblin", 40, 10);
    Skeleton skeleton("Skeleton", 60, 15);
    Dragon dragon("Dragon", 200, 50);

    // Массив указателей на базовый класс
    Character* monsters[3] = {&goblin, &skeleton, &dragon};
    int monsterCount = 3;

    std::cout << "=== Подземелье ===\n\n";

    for (int i = 0; i < monsterCount && knight.isAlive(); i++) {
        Character* monster = monsters[i];
        std::cout << "--- " << knight.getName() << " vs "
                  << monster->getName() << " ---\n";

        while (knight.isAlive() && monster->isAlive()) {
            knight.attack(*monster);

            if (monster->isAlive()) {
                // Каждый монстр атакует по-своему
                if (auto* g = dynamic_cast<Goblin*>(monster)) g->attack(knight);
                else if (auto* s = dynamic_cast<Skeleton*>(monster)) s->attack(knight);
                else if (auto* d = dynamic_cast<Dragon*>(monster)) d->attack(knight);
            }

            std::cout << knight << " | " << *monster << "\n";
        }

        std::cout << "\n";
    }

    if (knight.isAlive()) {
        std::cout << knight.getName() << " прошёл подземелье!\n";
    } else {
        std::cout << knight.getName() << " пал в бою...\n";
    }

    return 0;
}
```

## Шпаргалка

- Наследование: `class Player : public Character`
- `protected` — видно в производных классах, скрыто снаружи
- Конструктор базового класса вызывается в списке инициализации: `Player(...) : Character(n, h)`
- `Character::takeDamage(dmg)` — вызов метода базового класса
- Наследование = «является». Композиция = «имеет». Декоратор = «оборачивает».
- Если можно решить композицией — решай композицией

```cpp
class Character : public Fightable {
protected:
    std::string name;
    int hp;
public:
    Character(std::string n, int h) : name(n), hp(h) {}
    void takeDamage(int dmg) override { hp -= dmg; }
};

class Player : public Character {
    Striker& weapon;
public:
    Player(std::string n, int h, Striker& w) : Character(n, h), weapon(w) {}
};
```

## Напоследок

1. В Java все классы наследуют от `Object` — общий предок. В C++ такого нет — каждый класс сам по себе. Поэтому в C++ интерфейсы (абстрактные классы) важнее, чем в Java.

2. Создатели Go **убрали наследование** из языка полностью. Только интерфейсы и композиция. Rust поступил так же. Это говорит о том, что индустрия движется от наследования к композиции.

3. В 1990-х наследование считалось главным инструментом ООП. Строили иерархии по 10 уровней. Потом оказалось, что такой код невозможно менять. Книга «Design Patterns» (1994) сказала: «prefer composition over inheritance» — и это стало стандартным советом.

## Практика

### 10.1. Лекарь

Создай `Healer : public Character`. У лекаря есть мана. Метод `heal(Character& target)` восстанавливает HP цели (не выше максимума) и тратит ману.

### 10.2. Разбойник

Создай `Rogue : public Character` с шансом уклонения. Переопредели `takeDamage`: если «кубик» выпал удачно — урон не проходит.

### 10.3. Подземелье

Создай отряд из 3 героев (Player, Healer, Rogue) и 5 монстров (Goblin, Skeleton, Dragon и два обычных Character). Проведи бой: каждый ход каждый живой герой атакует текущего монстра. Лекарь лечит самого раненого героя вместо атаки. Монстр бьёт случайного живого героя.
