# 04. Объекты владеют объектами

В прошлом уроке мы создали `Player` и `Monster` — у обоих есть имя, здоровье, урон. Проблема: у `Player` урон хранится как обычное число. Но ведь урон наносит оружие, а не сам игрок. Кулаки — это одно, меч — другое, лук — третье. Где оружие?

## Когда полей становится слишком много

Добавим оружие как поля игрока:

```cpp
class Player {
private:
    std::string name;
    int hp;
    std::string weaponName;
    int weaponDamage;
};
```

Работает. Теперь добавим второе оружие для двуручного бойца:

```cpp
class Player {
private:
    std::string name;
    int hp;
    std::string weapon1Name;
    int weapon1Damage;
    std::string weapon2Name;
    int weapon2Damage;
};
```

Четыре поля на два оружия. А если у оружия появится скорость атаки? Шесть полей. Вес? Восемь. Три оружия? Двенадцать.

И ещё: когда нужно вывести оружие — пишешь `cout << weapon1Name << " (" << weapon1Damage << ")"`. Второе — то же самое. Третье — копируешь ещё раз.

Проблема не в количестве полей. Проблема в том, что оружие — это **отдельная сущность**, а мы размазали его по полям игрока.

## Оружие — отдельный объект

В уроке 03 мы уже создали класс `Weapon`. Он знает своё имя и урон, умеет себя показать. Пусть `Player` **владеет** объектом `Weapon`:

```cpp
class Weapon {
private:
    std::string name;
    int damage;

public:
    Weapon(std::string n, int d) : name(n), damage(d) {}

    int getDamage() const { return damage; }

    void print() const {
        std::cout << name << " (урон: " << damage << ")";
    }
};

class Player {
private:
    std::string name;
    int hp;
    Weapon weapon;

public:
    Player(std::string n, int h, Weapon w)
        : name(n), hp(h), weapon(w) {}

    void printStatus() const {
        std::cout << name << ": " << hp << " HP, оружие: ";
        weapon.print();
        std::cout << "\n";
    }
};
```

Теперь:

```cpp
Weapon sword("Sword", 25);
Player knight("Knight", 100, sword);
knight.printStatus(); // Knight: 100 HP, оружие: Sword (урон: 25)
```

Или в одну строку:

```cpp
Player knight("Knight", 100, Weapon("Sword", 25));
```

Одно оружие — один объект. Два — два. Оружие само знает, как себя вывести, какой у него урон. Игрок не дублирует эту логику — он просит оружие.

## Это и есть композиция

**Композиция** — когда один объект содержит другой объект как поле. Игрок **имеет** оружие. Оружие — самостоятельная сущность со своим поведением.

Отличие от «просто полей»:
- `int weaponDamage` — это данные. Они ничего не умеют.
- `Weapon weapon` — это объект. Он знает свой урон, умеет себя показать, может рассчитать урон с учётом бонусов.

Если сомневаешься, представь: «это штука, у которой есть имя и поведение?» Если да — это отдельный объект.

## Объект решает за себя

Игрок атакует другого. Кто считает урон? Раньше — игрок доставал число из поля и вычитал. Теперь — игрок просит оружие нанести удар:

```cpp
class Weapon {
private:
    std::string name;
    int damage;

public:
    Weapon(std::string n, int d) : name(n), damage(d) {}

    int strike() const {
        std::cout << name << " наносит удар!\n";
        return damage;
    }

    void print() const {
        std::cout << name << " (урон: " << damage << ")";
    }
};
```

А игрок использует оружие:

```cpp
void attack(Player& target) {
    int dmg = weapon.strike();
    target.takeDamage(dmg);
}
```

Игрок не знает, как оружие считает урон. Оружие не знает, как цель принимает урон. Каждый отвечает за себя.

## Замена оружия

Игрок находит новый меч. Как сменить оружие?

```cpp
class Player {
private:
    std::string name;
    int hp;
    Weapon weapon;

public:
    Player(std::string n, int h, Weapon w)
        : name(n), hp(h), weapon(w) {}

    void equip(Weapon newWeapon) {
        weapon = newWeapon;
    }

    // ... остальные методы
};
```

Заметь: `equip` принимает `Weapon newWeapon` без `&`. В уроке 03 мы узнали, что объекты лучше передавать по ссылке. Но здесь другой случай — мы хотим **сохранить свою копию** оружия. Если бы мы взяли ссылку, а оригинал потом исчез — у игрока было бы сломанное оружие.

```cpp
Player knight("Knight", 100, Weapon("Fist", 5));
knight.printStatus(); // Knight: 100 HP, оружие: Fist (урон: 5)

knight.equip(Weapon("Sword", 25));
knight.printStatus(); // Knight: 100 HP, оружие: Sword (урон: 25)
```

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
    Weapon() : name("Fist"), damage(5) {}  // нужен, потому что Player хранит Weapon как поле
    Weapon(std::string n, int d) : name(n), damage(d) {}

    int strike() const {
        return damage;
    }

    void print() const {
        std::cout << name << " (урон: " << damage << ")";
    }
};

class Player {
private:
    std::string name;
    int hp;
    Weapon weapon;

public:
    Player(std::string n, int h, Weapon w) : name(n), hp(h), weapon(w) {}

    void attack(Player& target) {
        int dmg = weapon.strike();
        std::cout << name << " атакует " << target.name << "!\n";
        target.takeDamage(dmg);
    }

    void takeDamage(int dmg) {
        hp -= dmg;
        if (hp < 0) hp = 0;
    }

    void equip(Weapon w) {
        weapon = w;
    }

    bool isAlive() const { return hp > 0; }

    void printStatus() const {
        std::cout << name << ": " << hp << " HP, оружие: ";
        weapon.print();
        std::cout << "\n";
    }
};

int main() {
    SetConsoleCP(1251);
    SetConsoleOutputCP(1251);

    Player knight("Knight", 100, Weapon("Sword", 25));
    Player rogue("Rogue", 80, Weapon("Dagger", 35));

    knight.printStatus();
    rogue.printStatus();
    std::cout << "\n";

    // Рыцарь находит топор!
    knight.equip(Weapon("Axe", 40));
    std::cout << "Knight нашёл Axe!\n";
    knight.printStatus();
    std::cout << "\n";

    // Бой
    int round = 1;
    while (knight.isAlive() && rogue.isAlive()) {
        std::cout << "--- Ход " << round << " ---\n";
        knight.attack(rogue);
        if (rogue.isAlive()) {
            rogue.attack(knight);
        }
        knight.printStatus();
        rogue.printStatus();
        round++;
    }

    std::cout << "\n";
    if (knight.isAlive()) {
        std::cout << "Knight победил!\n";
    } else {
        std::cout << "Rogue победил!\n";
    }

    return 0;
}
```

```text
Knight: 100 HP, оружие: Sword (урон: 25)
Rogue: 80 HP, оружие: Dagger (урон: 35)

Knight нашёл Axe!
Knight: 100 HP, оружие: Axe (урон: 40)

--- Ход 1 ---
Knight атакует Rogue!
Rogue атакует Knight!
Knight: 65 HP, оружие: Axe (урон: 40)
Rogue: 40 HP, оружие: Dagger (урон: 35)
--- Ход 2 ---
Knight атакует Rogue!
Knight: 65 HP, оружие: Axe (урон: 40)
Rogue: 0 HP, оружие: Dagger (урон: 35)

Knight победил!
```

## Шпаргалка

- Композиция — объект содержит другой объект как поле
- `Weapon weapon;` внутри `Player` — игрок **имеет** оружие
- Объект-поле инициализируется в списке инициализации: `Player(..., Weapon w) : weapon(w) {}`
- Каждый объект отвечает за себя: оружие знает свой урон, игрок — своё здоровье
- Замена объекта-поля: `weapon = newWeapon;`
- Если «эта штука имеет имя и поведение» — это отдельный объект

```cpp
class Weapon {
    std::string name;
    int damage;
public:
    Weapon(std::string n, int d) : name(n), damage(d) {}
    int strike() const { return damage; }
};

class Player {
    std::string name;
    int hp;
    Weapon weapon;  // Player ИМЕЕТ Weapon
public:
    Player(std::string n, int h, Weapon w) : name(n), hp(h), weapon(w) {}
};
```

## Напоследок

1. В Java всё устроено так же: объект как поле — основной способ строить сложные системы. В Python — аналогично: `self.weapon = Weapon("Sword", 25)`.

2. В индустрии есть известное правило: «предпочитай композицию наследованию» (prefer composition over inheritance). Мы ещё доберёмся до наследования — но заметь, что без него можно построить очень много.

3. В игровой индустрии архитектура Entity-Component-System (ECS) — это композиция, доведённая до абсолюта. Персонаж — это пустая сущность, к которой прикрепляют компоненты: здоровье, оружие, инвентарь, AI. Каждый компонент — маленький объект.

## Практика

### 4.1. Броня

Создай класс `Armor` с полями: название и защита.

Добавь броню как поле в `Player`. Когда игрок получает урон — броня уменьшает его. Метод `takeDamage` теперь учитывает броню.

Проверь:

```cpp
Player knight("Knight", 100, Weapon("Sword", 25), Armor("Iron Plate", 10));
knight.printStatus(); // Knight: 100 HP, Sword (урон: 25), Iron Plate (защита: 10)

knight.takeDamage(30);
knight.printStatus(); // Knight: 80 HP (30 - 10 = 20 урона прошло)
```

### 4.2. Зелье лечит игрока

Создай класс `Potion` с полями: название, сила лечения, количество зарядов.

Добавь метод `use(Player& target)` — зелье лечит игрока и тратит заряд. Когда заряды кончились — зелье не работает.

Проверь:

```cpp
Player hero("Knight", 100, Weapon("Sword", 25), Armor("Shield", 5));
Potion heal("Heal Potion", 30, 2);

hero.takeDamage(50);
hero.printStatus();    // Knight: 55 HP (50 - 5 защиты = 45 урона, 100 - 45 = 55)

heal.use(hero);
hero.printStatus();    // Knight: 85 HP

heal.use(hero);
hero.printStatus();    // Knight: 100 HP (не выше максимума)

heal.use(hero);        // Зелье закончилось!
```

Подсказка: чтобы HP не превышал максимум, тебе понадобится поле `maxHp` в `Player`. Задай его равным начальному HP в конструкторе.

Обрати внимание: зелье — отдельный объект со своим состоянием (заряды). Игрок не знает, как зелье работает. Зелье не знает, сколько у игрока HP максимум — оно просто лечит, а игрок сам следит за лимитом.

### 4.3. Сундук с лутом

Создай класс `Chest` с двумя полями: оружие и зелье.

При открытии сундук выдаёт свой лут — игрок получает оружие, а зелье используется сразу.

```cpp
Chest chest(Weapon("Fire Sword", 40), Potion("Elixir", 50, 1));
chest.open(hero);
hero.printStatus(); // у героя новое оружие + здоровье восстановлено
```

Подумай: сундук владеет оружием и зельем. После открытия — оружие переходит к игроку, зелье тратится. Каждый объект делает своё дело.
