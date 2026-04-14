# 13. Объекты обобщают

В прошлых уроках мы строили контейнеры вручную: массив игроков в `Party`, массив монстров в `main`. Каждый раз — одинаковая логика: добавить, удалить, найти. Но типы разные. Копировать код для каждого типа?

## Проблема: одинаковый код, разные типы

У нас есть отряд:

```cpp
class Party {
private:
    Player* members[10];
    int count;
public:
    void add(Player* p) { members[count++] = p; }
    Player* get(int i) { return members[i]; }
    int size() const { return count; }
};
```

Теперь нужен инвентарь — массив оружия:

```cpp
class Inventory {
private:
    Weapon* items[10];
    int count;
public:
    void add(Weapon* w) { items[count++] = w; }
    Weapon* get(int i) { return items[i]; }
    int size() const { return count; }
};
```

А ещё — список монстров, список зелий, список заклинаний... Код **идентичный**. Отличается только тип: `Player*`, `Weapon*`, `Monster*`.

Если найдёшь баг в `add` — чинить в пяти местах.

## Шаблон: тип как параметр

А что если сказать компилятору: «вот код, а тип подставь сам»?

```cpp
template <typename T>
class Container {
private:
    T* items[10];
    int count;

public:
    Container() : count(0) {}

    void add(T* item) {
        if (count < 10) {
            items[count++] = item;
        }
    }

    T* get(int i) const {
        if (i >= 0 && i < count) return items[i];
        return nullptr;
    }

    int size() const { return count; }
};
```

`template <typename T>` — «T — это какой-то тип, я скажу какой позже».

Теперь:

```cpp
Container<Player> party;
Container<Weapon> inventory;
Container<Monster> horde;
```

Один класс — три контейнера. Компилятор сам создаёт версию для каждого типа.

## Как это работает

Когда пишешь `Container<Player>`, компилятор **генерирует** класс, подставляя `Player` вместо `T`:

```cpp
// Компилятор создаёт это "за кулисами":
class Container_Player {
    Player* items[10];
    void add(Player* item) { ... }
    Player* get(int i) const { ... }
};
```

Это происходит при **компиляции**, не при запуске. Шаблон — это инструкция для компилятора, а не для процессора. Поэтому:

1. Шаблоны не стоят ничего в рантайме — работают так же быстро, как написанный вручную код
2. Весь код шаблона должен быть в `.h` файле — компилятору нужно видеть код при подстановке типа

## Шаблон функции

Шаблоны работают и для функций:

```cpp
template <typename T>
void printAll(Container<T>& container) {
    for (int i = 0; i < container.size(); i++) {
        std::cout << *container.get(i) << "\n";
    }
}
```

Работает для любого типа, у которого есть `operator<<`:

```cpp
Container<Player> party;
Container<Weapon> armory;

printAll(party);   // выводит игроков
printAll(armory);  // выводит оружие
```

## Ограничения: не любой тип подходит

Если шаблон использует `operator<<`, а тип его не имеет — ошибка компиляции:

```cpp
class Secret {
    int value;
};

Container<Secret> box;
Secret s;
box.add(&s);
printAll(box);  // ОШИБКА: Secret не имеет operator<<
```

Компилятор скажет, что для `Secret` нет `operator<<`. Шаблон работает с любым типом, **который поддерживает нужные операции**.

## Container с поиском

Добавим полезные методы:

```cpp
template <typename T>
class Container {
private:
    T* items[10];
    int count;

public:
    Container() : count(0) {}

    void add(T* item) {
        if (count < 10) {
            items[count++] = item;
        }
    }

    T* get(int i) const {
        if (i >= 0 && i < count) return items[i];
        return nullptr;
    }

    int size() const { return count; }

    // Удаление по индексу (сдвиг)
    void remove(int index) {
        if (index < 0 || index >= count) return;
        for (int i = index; i < count - 1; i++) {
            items[i] = items[i + 1];
        }
        count--;
    }

    // Поиск по имени (T должен иметь getName())
    T* findByName(const std::string& name) const {
        for (int i = 0; i < count; i++) {
            if (items[i]->getName() == name) {
                return items[i];
            }
        }
        return nullptr;
    }
};
```

`findByName` требует, чтобы у типа был метод `getName()`. У наших `Character` и `Weapon` он есть — значит, работает:

```cpp
Container<Character> team;
// ...
Character* knight = team.findByName("Knight");
```

## Полный пример

```cpp
#include <iostream>
#include <string>
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

    friend std::ostream& operator<<(std::ostream& out, const Weapon& w) {
        out << w.name << " (урон: " << w.damage << ")";
        return out;
    }
};

// === Шаблонный контейнер ===
template <typename T>
class Container {
private:
    T* items[20];
    int count;

public:
    Container() : count(0) {}

    void add(T* item) {
        if (count < 20) {
            items[count++] = item;
        }
    }

    T* get(int i) const {
        if (i >= 0 && i < count) return items[i];
        return nullptr;
    }

    int size() const { return count; }

    void remove(int index) {
        if (index < 0 || index >= count) return;
        for (int i = index; i < count - 1; i++) {
            items[i] = items[i + 1];
        }
        count--;
    }

    T* findByName(const std::string& name) const {
        for (int i = 0; i < count; i++) {
            if (items[i]->getName() == name) {
                return items[i];
            }
        }
        return nullptr;
    }
};

// === Шаблонная функция вывода ===
template <typename T>
void printAll(const Container<T>& container) {
    for (int i = 0; i < container.size(); i++) {
        std::cout << "  " << *container.get(i) << "\n";
    }
}

class Player : public Character {
private:
    Weapon* weapon;
public:
    Player(std::string n, int h, Weapon* w)
        : Character(n, h), weapon(w) {}

    void attack(Fightable& target) {
        int dmg = weapon->strike();
        std::cout << name << " атакует " << target.getName()
                  << " на " << dmg << "!\n";
        target.takeDamage(dmg);
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

int main() {
    srand(time(0));

    // Арсенал
    Weapon sword("Sword", 30);
    Weapon bow("Bow", 20);
    Weapon staff("Staff", 40);
    Weapon dagger("Dagger", 25);

    Container<Weapon> armory;
    armory.add(&sword);
    armory.add(&bow);
    armory.add(&staff);
    armory.add(&dagger);

    std::cout << "=== Арсенал ===\n";
    printAll(armory);

    // Поиск оружия
    Weapon* found = armory.findByName("Staff");
    if (found) {
        std::cout << "\nНайдено: " << *found << "\n";
    }

    // Отряд
    Player knight("Knight", 120, &sword);
    Player archer("Archer", 80, &bow);
    Player mage("Mage", 60, &staff);

    Container<Player> party;
    party.add(&knight);
    party.add(&archer);
    party.add(&mage);

    std::cout << "\n=== Отряд ===\n";
    printAll(party);

    // Орда монстров
    Monster goblin("Goblin", 40, 10);
    Monster troll("Troll", 100, 20);
    Monster dragon("Dragon", 200, 35);

    Container<Monster> horde;
    horde.add(&goblin);
    horde.add(&troll);
    horde.add(&dragon);

    std::cout << "\n=== Монстры ===\n";
    printAll(horde);

    // Бой: отряд vs орда
    std::cout << "\n=== Бой ===\n\n";

    for (int m = 0; m < horde.size() && party.size() > 0; m++) {
        Monster* monster = horde.get(m);
        std::cout << "--- " << monster->getName() << " выходит на бой! ---\n";

        while (monster->isAlive()) {
            // Каждый живой герой атакует
            for (int p = 0; p < party.size(); p++) {
                Player* player = party.get(p);
                if (player->isAlive()) {
                    player->attack(*monster);
                    if (!monster->isAlive()) break;
                }
            }

            // Монстр бьёт случайного героя
            if (monster->isAlive()) {
                int target = rand() % party.size();
                monster->attack(*party.get(target));
            }
        }

        std::cout << monster->getName() << " повержен!\n\n";

        // Убрать павших героев
        for (int p = party.size() - 1; p >= 0; p--) {
            if (!party.get(p)->isAlive()) {
                std::cout << party.get(p)->getName() << " пал в бою...\n";
                party.remove(p);
            }
        }
    }

    std::cout << "\n=== Выжившие ===\n";
    if (party.size() > 0) {
        printAll(party);
    } else {
        std::cout << "  Никто не выжил...\n";
    }

    return 0;
}
```

Один `Container<T>` — три контейнера для разных типов. Одна `printAll<T>` — для любого типа с `operator<<`. Никакого дублирования.

## Шпаргалка

- `template <typename T>` — параметризация типом
- `Container<Player>` — компилятор подставляет `Player` вместо `T`
- Шаблоны работают при компиляции, не в рантайме — нулевые накладные расходы
- Весь код шаблона — в `.h` файле
- Тип `T` должен поддерживать операции, которые шаблон использует
- Шаблоны работают и для классов, и для функций

```cpp
template <typename T>
class Container {
    T* items[20];
    int count;
public:
    void add(T* item) { items[count++] = item; }
    T* get(int i) const { return items[i]; }
    int size() const { return count; }
};

Container<Player> party;
Container<Weapon> armory;
```

## Напоследок

1. `std::vector` из стандартной библиотеки — это шаблонный контейнер, похожий на наш `Container`, только гораздо мощнее: динамический размер, итераторы, алгоритмы. `std::vector<Player>` — массив игроков, который растёт сам.

2. В Java шаблоны называются **дженерики** (generics): `ArrayList<Player>`. Но в Java тип стирается в рантайме (type erasure) — `ArrayList<Player>` и `ArrayList<Weapon>` внутри одинаковые. В C++ каждый `Container<Player>` и `Container<Weapon>` — это **отдельные классы**, сгенерированные компилятором. Поэтому C++ шаблоны быстрее, но увеличивают размер программы.

3. Шаблоны в C++ используются настолько интенсивно, что породили целое направление — **шаблонное метапрограммирование** (template metaprogramming). Можно писать программы, которые выполняются при компиляции: вычислять факториал, проверять типы, генерировать код. Это мощно, но читать такой код — испытание даже для опытных разработчиков.

## Практика

### 13.1. Контейнер с максимумом

Добавь в `Container` шаблонный метод `max()`, который возвращает указатель на элемент с наибольшим значением. Для сравнения используй метод, который ты определишь: `getValue()`.

Реализуй `getValue()` в `Player` (возвращает HP) и в `Weapon` (возвращает урон). Проверь: `party.max()` вернёт самого живучего, `armory.max()` — самое мощное оружие.

### 13.2. Сортировка контейнера

Добавь метод `sortByName()`, который сортирует элементы по имени (пузырьком — достаточно). Выведи арсенал до и после сортировки.

### 13.3. Лут-система

Создай шаблон `LootDrop<T>`: контейнер, из которого можно получить случайный предмет (`T* random()`). Монстры после смерти дропают лут:

- Гоблин дропает оружие: `LootDrop<Weapon>`
- Дракон дропает доспехи: `LootDrop<Armor>` (создай простой класс `Armor` с именем и защитой)

После каждого боя герой получает случайный предмет из лута побеждённого монстра.
