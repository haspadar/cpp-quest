# 15. Живой мир

Финальный проект. Всё, что ты изучил за курс — классы, композиция, инкапсуляция, декораторы, интерфейсы, наследование, стратегии, шаблоны, умные указатели — собирается в одну программу.

## Что строим

Dungeon Crawler — исследование подземелья:

- **Герой** с оружием, бронёй, инвентарём
- **Подземелье** из комнат — в каждой комнате монстр или сокровище
- **Бой** с разными стратегиями атаки и защиты
- **Лут** — монстры дропают оружие и зелья
- **Прокачка** — опыт, уровни, улучшение характеристик
- **Меню** — выбор действий в каждой комнате

## Архитектура

Каждый компонент — то, что мы уже строили:

| Компонент | Концепция из курса | Урок |
|---|---|---|
| `Weapon`, `Armor` | Композиция | 04 |
| `Player`, `Monster` → `Character` | Наследование | 10 |
| `Striker` | Интерфейс | 09 |
| `PoisonedWeapon`, `CriticalStrike` | Декоратор | 08 |
| `MeleeAttack`, `RangedAttack` | Стратегия | 11 |
| `Container<T>` | Шаблон | 13 |
| `unique_ptr<Weapon>` | RAII | 14 |
| Файлы `.h/.cpp` | Организация | 06 |
| Потоки, форматирование | Ввод-вывод | 12 |

## Структура файлов

```
dungeon/
├── main.cpp
├── Character.h / Character.cpp
├── Player.h / Player.cpp
├── Monster.h / Monster.cpp
├── Weapon.h / Weapon.cpp
├── Armor.h / Armor.cpp
├── Potion.h / Potion.cpp
├── Striker.h
├── Fightable.h
├── AttackStyle.h
├── Container.h          (шаблон — только .h)
├── Room.h / Room.cpp
├── Dungeon.h / Dungeon.cpp
└── Battle.h / Battle.cpp
```

Каждый класс — один файл. Шаблоны — только `.h`.

## Базовые классы

Они у тебя уже есть из предыдущих уроков:

```cpp
// Fightable.h
#pragma once
#include <string>

class Fightable {
public:
    virtual void takeDamage(int dmg) = 0;
    virtual bool isAlive() const = 0;
    virtual std::string getName() const = 0;
    virtual ~Fightable() {}
};
```

```cpp
// Striker.h
#pragma once

class Striker {
public:
    virtual int strike() const = 0;
    virtual ~Striker() {}
};
```

```cpp
// Character.h
#pragma once
#include "Fightable.h"
#include <iostream>

class Character : public Fightable {
protected:
    std::string name;
    int hp;
    int maxHp;
    int level;
    int xp;

public:
    Character(std::string n, int h)
        : name(n), hp(h), maxHp(h), level(1), xp(0) {}

    void takeDamage(int dmg) override {
        hp -= dmg;
        if (hp < 0) hp = 0;
    }

    void heal(int amount) {
        hp += amount;
        if (hp > maxHp) hp = maxHp;
    }

    bool isAlive() const override { return hp > 0; }
    std::string getName() const override { return name; }
    int getHp() const { return hp; }
    int getMaxHp() const { return maxHp; }
    int getLevel() const { return level; }

    void gainXp(int amount) {
        xp += amount;
        while (xp >= level * 100) {
            xp -= level * 100;
            level++;
            maxHp += 20;
            hp = maxHp;
            std::cout << name << " достиг уровня " << level << "!\n";
        }
    }

    friend std::ostream& operator<<(std::ostream& out, const Character& c) {
        out << c.name << " [Ур." << c.level << "] "
            << c.hp << "/" << c.maxHp << " HP";
        return out;
    }
};
```

## Предметы

```cpp
// Weapon.h
#pragma once
#include "Striker.h"
#include <string>
#include <iostream>

class Weapon : public Striker {
private:
    std::string name;
    int damage;

public:
    Weapon(std::string n, int d) : name(n), damage(d) {}

    int strike() const override { return damage; }
    std::string getName() const { return name; }
    int getDamage() const { return damage; }

    friend std::ostream& operator<<(std::ostream& out, const Weapon& w) {
        out << w.name << " (урон: " << w.damage << ")";
        return out;
    }
};
```

```cpp
// Armor.h
#pragma once
#include <string>
#include <iostream>

class Armor {
private:
    std::string name;
    int defense;

public:
    Armor(std::string n, int d) : name(n), defense(d) {}

    int reduce(int dmg) const {
        int actual = dmg - defense;
        return actual > 0 ? actual : 0;
    }

    std::string getName() const { return name; }
    int getDefense() const { return defense; }

    friend std::ostream& operator<<(std::ostream& out, const Armor& a) {
        out << a.name << " (защита: " << a.defense << ")";
        return out;
    }
};
```

```cpp
// Potion.h
#pragma once
#include <string>
#include <iostream>

class Potion {
private:
    std::string name;
    int healAmount;

public:
    Potion(std::string n, int h) : name(n), healAmount(h) {}

    int getHeal() const { return healAmount; }
    std::string getName() const { return name; }

    friend std::ostream& operator<<(std::ostream& out, const Potion& p) {
        out << p.name << " (+" << p.healAmount << " HP)";
        return out;
    }
};
```

## Игрок с инвентарём

```cpp
// Player.h
#pragma once
#include "Character.h"
#include "Weapon.h"
#include "Armor.h"
#include "Potion.h"
#include <memory>
#include <iomanip>

class Player : public Character {
private:
    std::unique_ptr<Weapon> weapon;
    std::unique_ptr<Armor> armor;
    std::unique_ptr<Potion> potions[5];
    int potionCount;

public:
    Player(std::string n, int h,
           std::unique_ptr<Weapon> w,
           std::unique_ptr<Armor> a)
        : Character(n, h),
          weapon(std::move(w)),
          armor(std::move(a)),
          potionCount(0) {}

    void takeDamage(int dmg) override {
        int actual = armor ? armor->reduce(dmg) : dmg;
        Character::takeDamage(actual);
        if (armor && actual < dmg) {
            std::cout << "  " << armor->getName()
                      << " блокирует " << (dmg - actual) << " урона\n";
        }
    }

    void attack(Fightable& target) {
        if (!weapon) {
            std::cout << name << " бьёт кулаком на 5!\n";
            target.takeDamage(5);
            return;
        }
        int dmg = weapon->strike();
        std::cout << name << " атакует " << target.getName()
                  << " [" << weapon->getName() << "] на " << dmg << "!\n";
        target.takeDamage(dmg);
    }

    void equip(std::unique_ptr<Weapon> w) {
        std::cout << name << " экипирует " << w->getName() << "\n";
        weapon = std::move(w);
    }

    void equip(std::unique_ptr<Armor> a) {
        std::cout << name << " надевает " << a->getName() << "\n";
        armor = std::move(a);
    }

    void addPotion(std::unique_ptr<Potion> p) {
        if (potionCount < 5) {
            std::cout << "Подобрано: " << *p << "\n";
            potions[potionCount++] = std::move(p);
        } else {
            std::cout << "Инвентарь зелий полон!\n";
        }
    }

    void usePotion() {
        if (potionCount == 0) {
            std::cout << "Нет зелий!\n";
            return;
        }
        potionCount--;
        int healAmt = potions[potionCount]->getHeal();
        heal(healAmt);
        std::cout << name << " выпивает " << potions[potionCount]->getName()
                  << "! +" << healAmt << " HP\n";
        potions[potionCount].reset();
    }

    void showStatus() const {
        std::cout << "\n=== " << name << " ===\n";
        std::cout << std::left;
        std::cout << std::setw(12) << "Уровень:" << level << "\n";
        std::cout << std::setw(12) << "HP:" << hp << "/" << maxHp << "\n";
        std::cout << std::setw(12) << "Оружие:"
                  << (weapon ? weapon->getName() : "кулаки") << "\n";
        std::cout << std::setw(12) << "Броня:"
                  << (armor ? armor->getName() : "нет") << "\n";
        std::cout << std::setw(12) << "Зелья:" << potionCount << "/5\n";
    }

    bool hasWeapon() const { return weapon != nullptr; }
    const Weapon* getWeapon() const { return weapon.get(); }
    const Armor* getArmor() const { return armor.get(); }
    int getPotionCount() const { return potionCount; }
};
```

## Монстры

```cpp
// Monster.h
#pragma once
#include "Character.h"
#include "Weapon.h"
#include "Armor.h"
#include "Potion.h"
#include <memory>

class Monster : public Character {
private:
    int damage;
    int xpReward;
    std::unique_ptr<Weapon> lootWeapon;
    std::unique_ptr<Armor> lootArmor;
    std::unique_ptr<Potion> lootPotion;

public:
    Monster(std::string n, int h, int d, int xpR)
        : Character(n, h), damage(d), xpReward(xpR) {}

    void setLoot(std::unique_ptr<Weapon> w) { lootWeapon = std::move(w); }
    void setLoot(std::unique_ptr<Armor> a) { lootArmor = std::move(a); }
    void setLoot(std::unique_ptr<Potion> p) { lootPotion = std::move(p); }

    void attack(Fightable& target) {
        std::cout << name << " бьёт " << target.getName()
                  << " на " << damage << "!\n";
        target.takeDamage(damage);
    }

    int getXpReward() const { return xpReward; }

    std::unique_ptr<Weapon> dropWeapon() { return std::move(lootWeapon); }
    std::unique_ptr<Armor> dropArmor() { return std::move(lootArmor); }
    std::unique_ptr<Potion> dropPotion() { return std::move(lootPotion); }
};
```

## Комната подземелья

```cpp
// Room.h
#pragma once
#include "Monster.h"
#include "Potion.h"
#include <memory>
#include <string>

enum class RoomType { MONSTER, TREASURE, EMPTY };

class Room {
private:
    std::string description;
    RoomType type;
    std::unique_ptr<Monster> monster;
    std::unique_ptr<Potion> treasure;

public:
    Room(std::string desc, std::unique_ptr<Monster> m)
        : description(desc), type(RoomType::MONSTER),
          monster(std::move(m)) {}

    Room(std::string desc, std::unique_ptr<Potion> t)
        : description(desc), type(RoomType::TREASURE),
          treasure(std::move(t)) {}

    Room(std::string desc)
        : description(desc), type(RoomType::EMPTY) {}

    RoomType getType() const { return type; }
    const std::string& getDescription() const { return description; }
    Monster* getMonster() { return monster.get(); }
    std::unique_ptr<Potion> takeTreasure() { return std::move(treasure); }
};
```

## Основной цикл: main.cpp

```cpp
#include <iostream>
#include <iomanip>
#include <memory>
#include <cstdlib>
#include <ctime>
#include <string>

// Все заголовки
#include "Room.h"
#include "Player.h"

// Бой
void battle(Player& player, Monster& monster) {
    std::cout << "\n  >>> БОЙ: " << player.getName()
              << " vs " << monster.getName() << " <<<\n\n";

    while (player.isAlive() && monster.isAlive()) {
        // Ход игрока
        std::cout << "  [1] Атака  [2] Зелье\n";
        std::cout << "  Выбор: ";
        int choice;
        std::cin >> choice;

        if (choice == 2 && player.getPotionCount() > 0) {
            player.usePotion();
        } else {
            player.attack(monster);
        }

        // Ход монстра
        if (monster.isAlive()) {
            monster.attack(player);
        }

        // Статус
        std::cout << "  " << player.getName() << ": "
                  << player.getHp() << "/" << player.getMaxHp() << " HP | "
                  << monster.getName() << ": "
                  << monster.getHp() << "/" << monster.getMaxHp() << " HP\n\n";
    }

    if (!monster.isAlive()) {
        std::cout << "  " << monster.getName() << " повержен!\n";

        // Опыт
        player.gainXp(monster.getXpReward());
        std::cout << "  +" << monster.getXpReward() << " XP\n";

        // Лут
        auto weapon = monster.dropWeapon();
        if (weapon) {
            if (!player.hasWeapon() ||
                weapon->getDamage() > player.getWeapon()->getDamage()) {
                player.equip(std::move(weapon));
            } else {
                std::cout << "  " << weapon->getName()
                          << " хуже текущего оружия.\n";
            }
        }

        auto armor = monster.dropArmor();
        if (armor) {
            if (!player.getArmor() ||
                armor->getDefense() > player.getArmor()->getDefense()) {
                player.equip(std::move(armor));
            } else {
                std::cout << "  " << armor->getName()
                          << " хуже текущей брони.\n";
            }
        }

        auto potion = monster.dropPotion();
        if (potion) {
            player.addPotion(std::move(potion));
        }
    }
}

// Создание подземелья
std::unique_ptr<Room> rooms[8];
int roomCount = 0;

void buildDungeon() {
    // Комната 1: слабый монстр
    auto goblin = std::make_unique<Monster>("Goblin", 30, 8, 30);
    goblin->setLoot(std::make_unique<Potion>("Small Potion", 20));
    rooms[roomCount++] = std::make_unique<Room>(
        "Тёмный коридор. Пахнет гнилью.", std::move(goblin));

    // Комната 2: сокровище
    rooms[roomCount++] = std::make_unique<Room>(
        "Заброшенная кладовая. На полке — зелье.",
        std::make_unique<Potion>("Medium Potion", 40));

    // Комната 3: монстр посильнее
    auto skeleton = std::make_unique<Monster>("Skeleton", 50, 12, 50);
    skeleton->setLoot(std::make_unique<Weapon>("Bone Sword", 20));
    rooms[roomCount++] = std::make_unique<Room>(
        "Зал с колоннами. Кости хрустят под ногами.", std::move(skeleton));

    // Комната 4: пусто
    rooms[roomCount++] = std::make_unique<Room>(
        "Пустая комната. Тишина.");

    // Комната 5: монстр с бронёй в дропе
    auto orc = std::make_unique<Monster>("Orc Warrior", 70, 15, 70);
    orc->setLoot(std::make_unique<Armor>("Chain Mail", 8));
    rooms[roomCount++] = std::make_unique<Room>(
        "Оружейная. Орк точит топор.", std::move(orc));

    // Комната 6: сокровище
    rooms[roomCount++] = std::make_unique<Room>(
        "Тайник за стеной. Внутри — зелье.",
        std::make_unique<Potion>("Large Potion", 60));

    // Комната 7: сильный монстр
    auto troll = std::make_unique<Monster>("Cave Troll", 120, 22, 100);
    troll->setLoot(std::make_unique<Weapon>("Troll Crusher", 35));
    troll->setLoot(std::make_unique<Potion>("Giant Potion", 80));
    rooms[roomCount++] = std::make_unique<Room>(
        "Огромная пещера. Земля дрожит от шагов.", std::move(troll));

    // Комната 8: босс
    auto dragon = std::make_unique<Monster>("Ancient Dragon", 200, 30, 200);
    dragon->setLoot(std::make_unique<Weapon>("Dragon Fang", 50));
    rooms[roomCount++] = std::make_unique<Room>(
        "Логово дракона. Жар обжигает лицо.", std::move(dragon));
}

int main() {
    srand(time(0));

    std::cout << "=== DUNGEON CRAWLER ===\n\n";
    std::cout << "Введи имя героя: ";
    std::string heroName;
    std::cin >> heroName;

    auto player = std::make_unique<Player>(
        heroName, 100,
        std::make_unique<Weapon>("Rusty Sword", 12),
        std::make_unique<Armor>("Leather Armor", 3)
    );

    buildDungeon();

    for (int i = 0; i < roomCount && player->isAlive(); i++) {
        Room* room = rooms[i].get();

        std::cout << "\n========================================\n";
        std::cout << "Комната " << (i + 1) << "/" << roomCount << "\n";
        std::cout << room->getDescription() << "\n";

        if (room->getType() == RoomType::MONSTER) {
            Monster* monster = room->getMonster();
            std::cout << "Перед тобой: " << *monster << "\n";

            std::cout << "\n[1] Сражаться  [2] Статус героя\n";
            std::cout << "Выбор: ";
            int choice;
            std::cin >> choice;

            if (choice == 2) {
                player->showStatus();
                std::cout << "\n[1] Сражаться\nВыбор: ";
                std::cin >> choice;
            }

            battle(*player, *monster);

        } else if (room->getType() == RoomType::TREASURE) {
            auto potion = room->takeTreasure();
            if (potion) {
                std::cout << "Найдено: " << *potion << "\n";
                player->addPotion(std::move(potion));
            }

        } else {
            std::cout << "Здесь пусто. Идём дальше.\n";
        }

        if (player->isAlive()) {
            std::cout << "\n" << *player << "\n";
        }
    }

    std::cout << "\n========================================\n";
    if (player->isAlive()) {
        std::cout << "\n*** ПОБЕДА! ***\n";
        player->showStatus();
    } else {
        std::cout << "\n*** ПОРАЖЕНИЕ ***\n";
        std::cout << heroName << " пал в подземелье...\n";
    }

    return 0;
}
```

## Что здесь используется

Пройдись по коду и найди каждую концепцию:

- **Классы** (урок 02): `Player`, `Monster`, `Weapon`, `Armor`, `Potion`, `Room`
- **Конструкторы** (урок 03): списки инициализации, `make_unique`
- **Композиция** (урок 04): Player имеет Weapon, Armor, Potion
- **Инкапсуляция** (урок 05): `takeDamage` скрывает расчёт брони, `heal` проверяет максимум
- **Файлы** (урок 06): каждый класс в своём `.h`
- **Наследование** (урок 10): Player и Monster → Character → Fightable
- **Потоки** (урок 12): `operator<<`, `setw`, `iomanip`
- **Умные указатели** (урок 14): `unique_ptr` для владения, `get()` для использования

## Что можно добавить

Проект живой — его можно развивать бесконечно:

1. **Декораторы оружия** (урок 08): найти отравленный или ледяной меч в подземелье
2. **Стратегии монстров** (урок 11): слабые монстры убегают при низком HP, боссы переходят в ярость
3. **Шаблонный контейнер** (урок 13): заменить массив зелий на `Container<Potion>`
4. **Сохранение/загрузка** (урок 12): записать прогресс в файл, загрузить при старте
5. **Случайная генерация**: процедурное подземелье — каждый запуск другое
6. **Магазин**: перед подземельем купить снаряжение за золото с монстров
7. **Отряд**: несколько героев с разными ролями (воин, маг, лекарь)

## Практика

### 15.1. Минимальная версия

Скомпилируй и запусти основной пример. Пройди подземелье. Проверь: работает ли лут, прокачка, зелья.

### 15.2. Расширение

Выбери **два** пункта из «что можно добавить» и реализуй их. Рекомендация:
- Если нравятся бои — добавь стратегии монстров и декораторы оружия
- Если нравится исследование — добавь случайную генерацию и сохранение

### 15.3. Свой проект

Перепиши подземелье на свой лад. Другая тематика (космос, пираты, зомби-апокалипсис), другие механики. Главное правило: **каждый объект — маленький и отвечает за одно**. Если класс больше 50 строк — подумай, не стоит ли его разделить.

Это не конец — это начало. Ты знаешь ООП. Дальше — практика.
