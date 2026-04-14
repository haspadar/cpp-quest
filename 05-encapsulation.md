# 05. Объекты сами за себя отвечают

В прошлом уроке мы собрали `Player` из `Weapon` и `Armor`. Объекты стали сложнее — и появились новые проблемы.

## Когда кто-то лезет не в своё дело

Посмотри на этот код:

```cpp
// Игрок лечится
int healAmount = 30;
int newHp = hero.hp + healAmount;
if (newHp > hero.maxHp) newHp = hero.maxHp;
hero.hp = newHp;
```

А теперь зелье лечит:

```cpp
int newHp = hero.hp + potion.power;
if (newHp > hero.maxHp) newHp = hero.maxHp;
hero.hp = newHp;
potion.charges--;
```

Один и тот же код проверки HP — в двух местах. Если добавится бонус к максимальному здоровью — менять оба. Если появится третий источник лечения — копировать ещё раз.

Проблема не в дублировании. Проблема в том, что **чужой код решает за игрока**, сколько у него здоровья. Игрок сам должен знать свой максимум и следить за ним.

## Скажи, не спрашивай

Вместо того чтобы спрашивать у объекта его данные и самому принимать решение — **скажи ему, что делать**:

```cpp
// Плохо: спрашиваем и решаем за него
int hp = hero.getHp();
int maxHp = hero.getMaxHp();
int newHp = hp + 30;
if (newHp > maxHp) newHp = maxHp;
hero.setHp(newHp);

// Хорошо: говорим что делать
hero.heal(30);
```

В первом случае внешний код знает про HP, максимум, ограничения. Во втором — всё это внутри `Player`. Если правила лечения изменятся (например, отравленный герой лечится на 50%) — менять только `Player::heal`.

## const: обещание не менять

Некоторые методы только читают данные, ничего не меняя. Компилятору можно об этом сказать — добавить `const` после скобок:

```cpp
class Player {
private:
    std::string name;
    int hp;
    int maxHp;

public:
    bool isAlive() const { return hp > 0; }     // не меняет объект
    void printStatus() const {                    // не меняет объект
        std::cout << name << ": " << hp << "/" << maxHp << " HP\n";
    }
    void takeDamage(int dmg) { hp -= dmg; }      // меняет — без const
};
```

Зачем? Если внутри `const`-метода случайно напишешь `hp = 0` — компилятор поймает ошибку. Это страховка.

Правило: если метод не меняет объект — ставь `const`.

## this: указатель на себя

Иногда нужно сослаться на текущий объект. Внутри метода `this` — указатель на объект, у которого вызван метод:

```cpp
class Player {
private:
    std::string name;
    int hp;

public:
    Player(std::string name, int hp) : name(name), hp(hp) {}
};
```

Здесь параметры `name` и `hp` совпадают с полями. Компилятор поймёт (в списке инициализации это работает), но иногда нужна явность:

```cpp
void rename(std::string name) {
    this->name = name;  // this->name — поле, name — параметр
}
```

## operator<<: объект сам знает, как себя показать

В уроке 04 мы выводили объекты методом `print()`:

```cpp
sword.print();
std::cout << "\n";
```

Но для чисел и строк мы используем `cout <<`. Хочется и для своих объектов:

```cpp
std::cout << sword << "\n";
```

Для этого нужно перегрузить оператор `<<`. Проблема: левая часть — `cout` (тип `ostream`), а не наш класс. Мы не можем добавить метод в чужой класс. Решение — **friend-функция**:

```cpp
class Weapon {
private:
    std::string name;
    int damage;

public:
    Weapon(std::string n, int d) : name(n), damage(d) {}

    friend std::ostream& operator<<(std::ostream& out, const Weapon& w) {
        out << w.name << " (урон: " << w.damage << ")";
        return out;
    }
};
```

- `friend` — функция не метод класса, но видит его private-поля
- `return out` — чтобы работали цепочки: `cout << sword << " готов!\n"`

Теперь:

```cpp
Weapon sword("Sword", 25);
std::cout << sword << "\n";           // Sword (урон: 25)
std::cout << "Оружие: " << sword;     // цепочка работает
```

## Когда геттеры нужны

«Скажи, не спрашивай» — хорошее правило. Но иногда нужно спросить: для вывода, для сравнения, для принятия решения **снаружи**.

Геттер — это метод, который возвращает значение поля:

```cpp
std::string getName() const { return name; }
int getHp() const { return hp; }
```

Правило: **геттер для чтения — нормально. Сеттер для записи — подумай дважды.** Если внешний код делает `player.setHp(player.getHp() - 10)` — значит, нужен метод `player.takeDamage(10)`.

## Полный пример

```cpp
#include <iostream>
#include <string>
#include <Windows.h>

class Weapon {
private:
    std::string name;
    int damage;

public:
    Weapon() : name("Fist"), damage(5) {}
    Weapon(std::string n, int d) : name(n), damage(d) {}

    int strike() const { return damage; }

    friend std::ostream& operator<<(std::ostream& out, const Weapon& w) {
        out << w.name << " (урон: " << w.damage << ")";
        return out;
    }
};

class Armor {
private:
    std::string name;
    int defense;

public:
    Armor() : name("None"), defense(0) {}
    Armor(std::string n, int d) : name(n), defense(d) {}

    int reduce(int dmg) const {
        int actual = dmg - defense;
        return actual > 0 ? actual : 0;
    }

    friend std::ostream& operator<<(std::ostream& out, const Armor& a) {
        out << a.name << " (защита: " << a.defense << ")";
        return out;
    }
};

class Player {
private:
    std::string name;
    int hp;
    int maxHp;
    Weapon weapon;
    Armor armor;

public:
    Player(std::string n, int h, Weapon w, Armor a)
        : name(n), hp(h), maxHp(h), weapon(w), armor(a) {}

    void attack(Player& target) const {
        int dmg = weapon.strike();
        std::cout << name << " атакует " << target.name << "!\n";
        target.takeDamage(dmg);
    }

    void takeDamage(int dmg) {
        int actual = armor.reduce(dmg);
        hp -= actual;
        if (hp < 0) hp = 0;
        std::cout << "  " << actual << " урона прошло\n";
    }

    void heal(int amount) {
        hp += amount;
        if (hp > maxHp) hp = maxHp;
    }

    void equip(Weapon w) { weapon = w; }

    bool isAlive() const { return hp > 0; }
    std::string getName() const { return name; }

    friend std::ostream& operator<<(std::ostream& out, const Player& p) {
        out << p.name << ": " << p.hp << "/" << p.maxHp << " HP, "
            << p.weapon << ", " << p.armor;
        return out;
    }
};

int main() {
    SetConsoleCP(1251);
    SetConsoleOutputCP(1251);

    Player knight("Knight", 100, Weapon("Sword", 25), Armor("Iron Plate", 10));
    Player rogue("Rogue", 80, Weapon("Dagger", 35), Armor("Leather", 5));

    std::cout << knight << "\n";
    std::cout << rogue << "\n\n";

    int round = 1;
    while (knight.isAlive() && rogue.isAlive()) {
        std::cout << "--- Ход " << round << " ---\n";
        knight.attack(rogue);
        if (rogue.isAlive()) {
            rogue.attack(knight);
        }
        std::cout << knight << "\n";
        std::cout << rogue << "\n";
        round++;
    }

    std::cout << "\n";
    if (knight.isAlive()) {
        std::cout << knight.getName() << " победил!\n";
    } else {
        std::cout << rogue.getName() << " победил!\n";
    }

    return 0;
}
```

```text
Knight: 100/100 HP, Sword (урон: 25), Iron Plate (защита: 10)
Rogue: 80/80 HP, Dagger (урон: 35), Leather (защита: 5)

--- Ход 1 ---
Knight атакует Rogue!
  20 урона прошло
Rogue атакует Knight!
  25 урона прошло
Knight: 75/100 HP, Sword (урон: 25), Iron Plate (защита: 10)
Rogue: 60/80 HP, Dagger (урон: 35), Leather (защита: 5)
...
```

Каждый объект отвечает за себя: `Armor::reduce` считает поглощение, `Player::takeDamage` применяет его, `operator<<` выводит — и всё это не размазано по `main()`.

## Шпаргалка

- «Скажи, не спрашивай»: `hero.heal(30)` вместо `hero.setHp(hero.getHp() + 30)`
- `const` после метода — обещание не менять объект: `bool isAlive() const`
- `this` — указатель на текущий объект: `this->name = name`
- `friend ostream& operator<<` — вывод через `cout <<`
- `return out` — чтобы работали цепочки `<< ... << ...`
- Геттеры для чтения — нормально. Сеттеры — подумай, не нужен ли метод с поведением

```cpp
class Player {
    std::string name;
    int hp;
public:
    bool isAlive() const { return hp > 0; }
    void heal(int amount) { hp += amount; if (hp > maxHp) hp = maxHp; }
    friend std::ostream& operator<<(std::ostream& out, const Player& p) {
        out << p.name << ": " << p.hp << " HP";
        return out;
    }
};
```

## Напоследок

1. Принцип «Tell, Don't Ask» сформулировал Мартин Фаулер. Но идея старше — Алан Кей, создатель ООП, говорил: «Я придумал объектно-ориентированное программирование, и C++ — это не то, что я имел в виду». Он имел в виду объекты, которые общаются сообщениями, а не объекты, из которых вытаскивают данные.

2. В Python `operator<<` нет — вместо него `__str__`: метод, который возвращает строку. `print(player)` вызывает `player.__str__()`. В Java — `toString()`. Идея та же: объект сам решает, как себя показать.

3. Ключевое слово `friend` — уникальная особенность C++. В Java и Python его нет. Там для `operator<<` просто делают метод `toString()`. В C++ пришлось изобрести `friend`, потому что `cout` — не наш класс, а мы хотим научить его выводить наши объекты.

## Практика

### 5.1. Монстр с оператором вывода

Создай класс `Monster` с полями: имя, HP, урон. Перегрузи `operator<<`, чтобы работало:

```cpp
Monster goblin("Goblin", 60, 15);
std::cout << goblin << "\n"; // Goblin: 60 HP, урон: 15
```

Добавь метод `attack(Player& target)` — монстр бьёт игрока. И проведи бой:

```cpp
Player hero("Knight", 100, Weapon("Sword", 25), Armor("Iron Plate", 10));
Monster goblin("Goblin", 60, 15);

while (hero.isAlive() && goblin.isAlive()) {
    // герой бьёт монстра, монстр бьёт героя
    std::cout << hero << "\n" << goblin << "\n---\n";
}
```

### 5.2. Зелье с зарядами

Создай класс `Potion` с `operator<<`. Зелье показывает название, силу и оставшиеся заряды:

```cpp
Potion heal("Heal", 30, 3);
std::cout << heal << "\n"; // Heal (сила: 30, заряды: 3)
```

Метод `use(Player& target)` лечит игрока и тратит заряд. Если заряды кончились — выводит сообщение и не лечит.

Обрати внимание: зелье не делает `target.setHp(...)` — оно вызывает `target.heal(power)`. Игрок сам следит за максимумом.

### 5.3. Лог боя

Создай бой: Knight (100 HP, Sword, Iron Plate) vs Goblin (60 HP, 15 урона). У Knight есть 2 зелья (Heal, сила 25, 1 заряд каждое).

Правила: каждый ход Knight бьёт Goblin. Если HP Knight ниже 40 и есть зелье — использует зелье вместо атаки.

Выводи каждый ход через `operator<<` — чтобы `main()` был чистым:

```cpp
std::cout << "Ход " << round << ": " << hero << " vs " << goblin << "\n";
```
