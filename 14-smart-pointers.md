# 14. Объекты владеют эксклюзивно

В прошлых уроках мы использовали указатели: `Container` хранит `T*`, `Player` принимает `Weapon*`. Всё работает, пока мы не начинаем создавать объекты динамически.

## Проблема: кто удаляет?

В лут-системе монстр дропает оружие. Мы создаём его через `new`:

```cpp
Weapon* loot = new Weapon("Dragon Sword", 50);
player.equip(loot);
```

Кто вызовет `delete`? Игрок? Когда? А если игрок умрёт? А если оружие передали другому игроку?

```cpp
void dungeon() {
    Weapon* sword = new Weapon("Sword", 30);
    Player* knight = new Player("Knight", 100, sword);

    // бой, бой, бой...

    // Кто удалит sword? Кто удалит knight?
    // Если knight удалит sword в деструкторе — а вдруг sword ещё нужен?
    // Если никто не удалит — утечка памяти.
}
// sword и knight утекли навсегда
```

Утечка памяти — программа жрёт всё больше памяти, пока не упадёт. В играх это приводит к лагам через час игры.

## Правило: у каждого ресурса — один владелец

В реальной жизни: у меча один хозяин. Хозяин умер — меч лежит на земле (или уничтожен вместе с ним). Два рыцаря не могут одновременно «владеть» одним мечом.

В C++ то же самое: **один объект владеет другим и отвечает за его удаление**.

## unique_ptr: эксклюзивный владелец

`std::unique_ptr` — умный указатель, который:
1. Хранит указатель на объект
2. **Автоматически удаляет** объект, когда сам уничтожается
3. **Нельзя копировать** — только передать владение

```cpp
#include <memory>

std::unique_ptr<Weapon> sword = std::make_unique<Weapon>("Sword", 30);
// sword — единственный владелец

std::cout << sword->strike() << "\n";  // используем как обычный указатель

// unique_ptr нельзя копировать:
// auto copy = sword;  // ОШИБКА!

// Но можно передать владение:
auto newOwner = std::move(sword);
// теперь newOwner владеет мечом, sword == nullptr
```

Когда `unique_ptr` выходит из области видимости — объект автоматически удаляется. Не нужен `delete`. Не нужно помнить. Не бывает утечек.

## RAII: ресурс привязан к жизни объекта

Этот принцип называется **RAII** — Resource Acquisition Is Initialization. Ресурс (память, файл, соединение) захватывается при создании объекта и освобождается при его уничтожении.

Ты уже видел RAII:

```cpp
std::ofstream file("report.txt");  // файл открыт при создании
file << "данные";
// file выходит из области видимости → файл закрыт автоматически
```

`unique_ptr` — тот же принцип для динамической памяти.

## Player владеет оружием

```cpp
class Player : public Character {
private:
    std::unique_ptr<Weapon> weapon;

public:
    Player(std::string n, int h, std::unique_ptr<Weapon> w)
        : Character(n, h), weapon(std::move(w)) {}

    void equip(std::unique_ptr<Weapon> newWeapon) {
        weapon = std::move(newWeapon);
        // старое оружие автоматически удалено!
    }

    void attack(Fightable& target) {
        if (!weapon) {
            std::cout << name << " безоружен!\n";
            return;
        }
        int dmg = weapon->strike();
        std::cout << name << " атакует " << target.getName()
                  << " на " << dmg << "!\n";
        target.takeDamage(dmg);
    }

    // Отдать оружие (потерять владение)
    std::unique_ptr<Weapon> dropWeapon() {
        return std::move(weapon);
        // weapon теперь nullptr
    }
};
```

Ключевое:
- `std::move` **передаёт** владение. После `std::move(w)` старая переменная пуста (`nullptr`).
- `equip` принимает `unique_ptr` по значению — **забирает** владение у вызывающего.
- `dropWeapon` **отдаёт** владение вызывающему.
- Когда `Player` умирает — его `weapon` автоматически удаляется.

## Создание объектов

```cpp
// make_unique создаёт объект и оборачивает в unique_ptr
auto sword = std::make_unique<Weapon>("Sword", 30);
auto knight = std::make_unique<Player>("Knight", 120, std::move(sword));

// или прямо в аргументе:
auto mage = std::make_unique<Player>(
    "Mage", 60,
    std::make_unique<Weapon>("Staff", 40)
);
```

`std::make_unique<T>(аргументы...)` — создаёт `T` в куче и возвращает `unique_ptr<T>`.

## Передача без владения: обычные указатели и ссылки

Не всегда нужно передавать владение. Часто функция просто **использует** объект:

```cpp
// Функция не забирает владение — принимает сырой указатель
void printWeapon(const Weapon* w) {
    if (w) std::cout << *w << "\n";
}

// Или ссылку
void useWeapon(Weapon& w) {
    std::cout << "Урон: " << w.strike() << "\n";
}

auto sword = std::make_unique<Weapon>("Sword", 30);
printWeapon(sword.get());   // .get() возвращает сырой указатель
useWeapon(*sword);          // разыменование даёт ссылку
```

Правило:
- **`unique_ptr`** — для владения: «я создал, я удалю»
- **Сырой указатель / ссылка** — для использования: «я только посмотрю»

## Полный пример

```cpp
#include <iostream>
#include <string>
#include <memory>
#include <cstdlib>
#include <ctime>

class Fightable {
public:
    virtual void takeDamage(int dmg) = 0;
    virtual bool isAlive() const = 0;
    virtual std::string getName() const = 0;
    virtual ~Fightable() {}
};

class Character : public Fightable {
protected:
    std::string name;
    int hp;
    int maxHp;
public:
    Character(std::string n, int h) : name(n), hp(h), maxHp(h) {}
    void takeDamage(int dmg) override { hp -= dmg; if (hp < 0) hp = 0; }
    bool isAlive() const override { return hp > 0; }
    std::string getName() const override { return name; }

    friend std::ostream& operator<<(std::ostream& out, const Character& c) {
        out << c.name << ": " << c.hp << "/" << c.maxHp << " HP";
        return out;
    }
};

class Weapon {
private:
    std::string name;
    int damage;
public:
    Weapon(std::string n, int d) : name(n), damage(d) {}
    int strike() const { return damage; }
    std::string getName() const { return name; }
    int getDamage() const { return damage; }

    friend std::ostream& operator<<(std::ostream& out, const Weapon& w) {
        out << w.name << " (урон: " << w.damage << ")";
        return out;
    }
};

class Player : public Character {
private:
    std::unique_ptr<Weapon> weapon;

public:
    Player(std::string n, int h, std::unique_ptr<Weapon> w)
        : Character(n, h), weapon(std::move(w)) {}

    void equip(std::unique_ptr<Weapon> newWeapon) {
        std::cout << name << " экипирует " << newWeapon->getName() << "!\n";
        weapon = std::move(newWeapon);
    }

    std::unique_ptr<Weapon> dropWeapon() {
        std::cout << name << " бросает " << weapon->getName() << "!\n";
        return std::move(weapon);
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

    bool hasWeapon() const { return weapon != nullptr; }

    const Weapon* getWeapon() const { return weapon.get(); }
};

class Monster : public Character {
private:
    int damage;
    std::unique_ptr<Weapon> loot;  // дроп после смерти

public:
    Monster(std::string n, int h, int d, std::unique_ptr<Weapon> drop)
        : Character(n, h), damage(d), loot(std::move(drop)) {}

    void attack(Fightable& target) {
        std::cout << name << " бьёт " << target.getName()
                  << " на " << damage << "!\n";
        target.takeDamage(damage);
    }

    // Отдать лут победителю
    std::unique_ptr<Weapon> dropLoot() {
        if (loot) {
            std::cout << name << " роняет " << loot->getName() << "!\n";
        }
        return std::move(loot);
    }
};

int main() {
    srand(time(0));

    // Создаём героя с мечом
    auto knight = std::make_unique<Player>(
        "Knight", 120,
        std::make_unique<Weapon>("Iron Sword", 25)
    );

    // Создаём монстров с лутом
    auto goblin = std::make_unique<Monster>(
        "Goblin", 40, 10,
        std::make_unique<Weapon>("Rusty Dagger", 15)
    );

    auto troll = std::make_unique<Monster>(
        "Troll", 80, 20,
        std::make_unique<Weapon>("War Hammer", 35)
    );

    auto dragon = std::make_unique<Monster>(
        "Dragon", 150, 30,
        std::make_unique<Weapon>("Dragon Fang", 50)
    );

    // Массив монстров (сырые указатели — не владеем)
    Monster* monsters[] = {goblin.get(), troll.get(), dragon.get()};
    int monsterCount = 3;

    std::cout << "=== Подземелье ===\n\n";
    std::cout << *knight << "\n";
    if (knight->hasWeapon()) {
        std::cout << "Оружие: " << *knight->getWeapon() << "\n";
    }
    std::cout << "\n";

    for (int i = 0; i < monsterCount && knight->isAlive(); i++) {
        Monster* monster = monsters[i];
        std::cout << "--- " << knight->getName() << " vs "
                  << monster->getName() << " ---\n";

        while (knight->isAlive() && monster->isAlive()) {
            knight->attack(*monster);
            if (monster->isAlive()) {
                monster->attack(*knight);
            }
        }

        if (!monster->isAlive()) {
            std::cout << monster->getName() << " повержен!\n";

            // Забираем лут
            auto loot = monster->dropLoot();
            if (loot) {
                // Берём, если лут лучше текущего оружия
                if (!knight->hasWeapon() ||
                    loot->getDamage() > knight->getWeapon()->getDamage()) {
                    knight->equip(std::move(loot));
                } else {
                    std::cout << "Лут хуже текущего оружия, пропускаем.\n";
                    // loot автоматически удалится здесь
                }
            }
        }

        std::cout << *knight << "\n\n";
    }

    if (knight->isAlive()) {
        std::cout << knight->getName() << " прошёл подземелье!\n";
        if (knight->hasWeapon()) {
            std::cout << "Финальное оружие: " << *knight->getWeapon() << "\n";
        }
    } else {
        std::cout << knight->getName() << " пал в бою...\n";
    }

    // Все unique_ptr автоматически очищаются при выходе из main
    return 0;
}
```

Ни одного `delete`. Ни одной утечки. Владение чёткое: Player владеет Weapon, Monster владеет лутом. Передача — через `std::move`.

## Шпаргалка

- `std::unique_ptr<T>` — эксклюзивный владелец объекта в куче
- `std::make_unique<T>(args...)` — создать объект и обернуть в `unique_ptr`
- `std::move(ptr)` — передать владение (старый указатель становится `nullptr`)
- `ptr.get()` — получить сырой указатель (без передачи владения)
- `*ptr` — разыменовать, `ptr->method()` — вызвать метод
- RAII: ресурс привязан к жизни объекта
- Владение = `unique_ptr`. Использование = сырой указатель / ссылка.

```cpp
auto sword = std::make_unique<Weapon>("Sword", 30);

Player knight("Knight", 100, std::move(sword));
// sword теперь nullptr, knight владеет мечом

auto loot = knight.dropWeapon();
// knight безоружен, loot владеет мечом
```

## Напоследок

1. В Java и Python **сборщик мусора** (garbage collector) сам находит и удаляет ненужные объекты. Удобно, но непредсказуемо: GC может запуститься в любой момент и заморозить программу на миллисекунды. В играх это проявляется как микрофризы. В C++ деструктор вызывается **детерминированно** — ты точно знаешь, когда объект умрёт.

2. Rust пошёл дальше C++: там владение — часть **системы типов**. Компилятор Rust **не скомпилирует** код, если владение неоднозначно. `unique_ptr` в C++ — добровольное соглашение. В Rust — принудительное правило.

3. Есть ещё `std::shared_ptr` — указатель с подсчётом ссылок. Несколько `shared_ptr` могут указывать на один объект, он удаляется, когда последний `shared_ptr` уничтожен. Это удобно, но медленнее `unique_ptr` и маскирует нечёткое владение. Хорошее правило: **начинай с `unique_ptr`, переходи на `shared_ptr` только если `unique_ptr` не подходит**.

## Практика

### 14.1. Инвентарь на unique_ptr

Создай класс `Inventory`, который хранит массив `std::unique_ptr<Weapon>`. Методы:
- `add(std::unique_ptr<Weapon> w)` — добавить оружие
- `remove(int index)` — удалить (автоматически!)
- `best()` — вернуть указатель (не владеющий) на лучшее оружие

### 14.2. Торговля

Два игрока. Один передаёт (`std::move`) оружие другому. Проверь: у первого оружия нет, у второго — есть. Попробуй атаковать без оружия.

### 14.3. Подземелье с лутом

Три комнаты, в каждой монстр с лутом (`unique_ptr<Weapon>`). После победы герой сравнивает лут с текущим оружием и берёт лучшее. Старое оружие автоматически удаляется. В конце — вывести финальную экипировку.
