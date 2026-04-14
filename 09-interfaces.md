# 09. Объекты дают обещания

В прошлом уроке мы столкнулись с проблемой: `PoisonedWeapon` нельзя обернуть в `CriticalStrike`, потому что `CriticalStrike` принимает `Weapon&`, а не `PoisonedWeapon&`. Это разные типы, хотя делают одно и то же — `strike()`.

## Проблема: все умеют бить, но тип разный

У нас три класса:

```cpp
Weapon         → int strike() const;
PoisonedWeapon → int strike() const;
CriticalStrike → int strike() const;
```

Все умеют `strike()`. Но `Player` принимает только `Weapon`:

```cpp
class Player {
    Weapon weapon;  // только Weapon, не PoisonedWeapon
};
```

Хочется сказать: «мне всё равно, что это за объект — лишь бы он умел `strike()`».

## Интерфейс: обещание уметь

В C++ можно создать класс, который описывает **что объект умеет**, не говоря **как**:

```cpp
class Striker {
public:
    virtual int strike() const = 0;
    virtual ~Striker() {}
};
```

`virtual int strike() const = 0` — **чисто виртуальный метод**. `= 0` означает: «здесь нет реализации. Каждый, кто наследует `Striker`, обязан написать свою».

`Striker` — это обещание: «я умею бить». Не важно как — мечом, ядом, критом. Важно, что метод `strike()` существует.

## Выполняем обещание

Теперь каждый класс «подписывается» под обещанием:

```cpp
class Weapon : public Striker {
private:
    std::string name;
    int damage;

public:
    Weapon(std::string n, int d) : name(n), damage(d) {}

    int strike() const override {
        return damage;
    }
};

class PoisonedWeapon : public Striker {
private:
    Striker& base;
    int poisonDmg;

public:
    PoisonedWeapon(Striker& w, int poison)
        : base(w), poisonDmg(poison) {}

    int strike() const override {
        return base.strike() + poisonDmg;
    }
};

class CriticalStrike : public Striker {
private:
    Striker& base;
    int critChance;

public:
    CriticalStrike(Striker& w, int chance)
        : base(w), critChance(chance) {}

    int strike() const override {
        int dmg = base.strike();
        if (rand() % 100 < critChance) dmg *= 2;
        return dmg;
    }
};
```

Заметь: `PoisonedWeapon` принимает `Striker&`, не `Weapon&`. И `CriticalStrike` тоже. Теперь обёртки можно комбинировать:

```cpp
Weapon sword("Sword", 25);
PoisonedWeapon poisoned(sword, 10);       // яд на мече
CriticalStrike critPoison(poisoned, 30);  // крит на отравленном мече!

int dmg = critPoison.strike(); // (25 + 10) * 2 = 70 при крите
```

Матрёшка работает на любую глубину. Каждый слой видит только `Striker&` — ему не важно, что внутри.

## virtual и override

Два ключевых слова:

- **`virtual`** — метод, который может быть переопределён в производных классах. Выбирается по реальному типу объекта, а не по типу указателя/ссылки.
- **`override`** — «я переопределяю виртуальный метод». Если опечатался в имени — компилятор поймает.

```cpp
class Striker {
public:
    virtual int strike() const = 0;  // чисто виртуальный
    virtual ~Striker() {}            // виртуальный деструктор
};

class Weapon : public Striker {
public:
    int strike() const override {    // переопределяем
        return damage;
    }
};
```

`= 0` делает класс **абстрактным** — нельзя создать объект `Striker`, можно только наследовать от него.

```cpp
Striker s;  // ошибка! Абстрактный класс
```

## Виртуальный деструктор

Если удаляешь объект через указатель на базовый класс — деструктор должен быть `virtual`:

```cpp
Striker* weapon = new Weapon("Sword", 25);
delete weapon; // без virtual ~Striker() — деструктор Weapon не вызовется!
```

Правило: **есть хотя бы один `virtual` метод — деструктор тоже `virtual`**.

## Player принимает любое оружие

Теперь `Player` хранит не `Weapon`, а ссылку на `Striker`:

```cpp
class Player {
private:
    std::string name;
    int hp;
    Striker& weapon;

public:
    Player(std::string n, int h, Striker& w)
        : name(n), hp(h), weapon(w) {}

    void attack(Player& target) {
        int dmg = weapon.strike();
        std::cout << name << " атакует " << target.name
                  << " на " << dmg << " урона!\n";
        target.takeDamage(dmg);
    }

    void takeDamage(int dmg) {
        hp -= dmg;
        if (hp < 0) hp = 0;
    }

    bool isAlive() const { return hp > 0; }
};
```

Игроку всё равно, что у него в руках — меч, отравленный меч или отравленный меч с критом. Он знает одно: оружие умеет `strike()`.

```cpp
Weapon sword("Sword", 25);
PoisonedWeapon venom(sword, 10);
CriticalStrike critVenom(venom, 20);

Player knight("Knight", 100, critVenom);  // отравленный крит-меч
Player rogue("Rogue", 80, sword);         // обычный меч

knight.attack(rogue); // использует всю цепочку: крит → яд → меч
```

## Это и есть полиморфизм

**Полиморфизм** — «много форм». Один вызов `weapon.strike()`, но результат зависит от реального объекта: обычный меч, отравленный, с критом.

Код `Player::attack` написан один раз. Он работает с любым оружием, которое ещё не создано. Добавишь `FrostWeapon` через неделю — `Player` менять не нужно.

## Полный пример

```cpp
#include <iostream>
#include <string>
#include <cstdlib>
#include <ctime>
#include <Windows.h>

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

    int strike() const override {
        return damage;
    }

    friend std::ostream& operator<<(std::ostream& out, const Weapon& w) {
        out << w.name << " (урон: " << w.damage << ")";
        return out;
    }
};

class PoisonedWeapon : public Striker {
private:
    Striker& base;
    int poisonDmg;

public:
    PoisonedWeapon(Striker& w, int poison)
        : base(w), poisonDmg(poison) {}

    int strike() const override {
        int dmg = base.strike();
        std::cout << "  Яд: +" << poisonDmg << "\n";
        return dmg + poisonDmg;
    }
};

class CriticalStrike : public Striker {
private:
    Striker& base;
    int critChance;

public:
    CriticalStrike(Striker& w, int chance)
        : base(w), critChance(chance) {}

    int strike() const override {
        int dmg = base.strike();
        if (rand() % 100 < critChance) {
            std::cout << "  Критический удар!\n";
            dmg *= 2;
        }
        return dmg;
    }
};

class Player {
private:
    std::string name;
    int hp;
    int maxHp;
    Striker& weapon;

public:
    Player(std::string n, int h, Striker& w)
        : name(n), hp(h), maxHp(h), weapon(w) {}

    void attack(Player& target) {
        std::cout << name << " атакует " << target.name << ":\n";
        int dmg = weapon.strike();
        std::cout << "  Итого: " << dmg << " урона\n";
        target.takeDamage(dmg);
    }

    void takeDamage(int dmg) {
        hp -= dmg;
        if (hp < 0) hp = 0;
    }

    bool isAlive() const { return hp > 0; }

    friend std::ostream& operator<<(std::ostream& out, const Player& p) {
        out << p.name << ": " << p.hp << "/" << p.maxHp << " HP";
        return out;
    }
};

int main() {
    SetConsoleCP(1251);
    SetConsoleOutputCP(1251);
    srand(time(0));

    // Knight: меч с ядом и критом
    Weapon sword("Sword", 25);
    PoisonedWeapon poisonSword(sword, 10);
    CriticalStrike critPoisonSword(poisonSword, 25);

    // Rogue: обычный кинжал
    Weapon dagger("Dagger", 35);

    Player knight("Knight", 100, critPoisonSword);
    Player rogue("Rogue", 80, dagger);

    std::cout << knight << "\n" << rogue << "\n\n";

    int round = 1;
    while (knight.isAlive() && rogue.isAlive()) {
        std::cout << "--- Ход " << round << " ---\n";
        knight.attack(rogue);
        if (rogue.isAlive()) {
            rogue.attack(knight);
        }
        std::cout << knight << "\n" << rogue << "\n\n";
        round++;
    }

    if (knight.isAlive()) {
        std::cout << "Knight победил!\n";
    } else {
        std::cout << "Rogue победил!\n";
    }

    return 0;
}
```

## Шпаргалка

- `virtual` — метод выбирается по реальному типу объекта
- `= 0` — чисто виртуальный метод, делает класс абстрактным
- `override` — помечает переопределение (защита от опечаток)
- Абстрактный класс нельзя создать — только наследовать от него
- Интерфейс = класс, где все методы `= 0`
- Виртуальный деструктор: `virtual ~Striker() {}`
- Полиморфизм: один вызов, разное поведение
- Декораторы + интерфейсы = комбинируемые обёртки любой глубины

```cpp
class Striker {
public:
    virtual int strike() const = 0;
    virtual ~Striker() {}
};

class Weapon : public Striker {
public:
    int strike() const override { return damage; }
};

class Poison : public Striker {
    Striker& base;
public:
    int strike() const override { return base.strike() + extra; }
};
```

## Напоследок

1. В Java вместо абстрактного класса с `= 0` есть отдельное ключевое слово `interface`. В C++ интерфейс — это просто класс, где все методы чисто виртуальные. Результат тот же, синтаксис разный.

2. В Go и TypeScript интерфейсы «утиные»: если объект имеет метод `strike()` — он автоматически реализует интерфейс `Striker`. Не нужно писать `: public Striker`. В C++ привязка явная — пишешь `: public Striker` или ничего не работает.

3. Слово «полиморфизм» с греческого — «много форм». Один и тот же вызов `strike()`, но за ним может стоять меч, яд, крит или любая их комбинация. Термин придумали не программисты, а биологи — для описания организмов, которые меняют форму.

## Практика

### 9.1. Ледяное оружие

Создай обёртку `FrostWeapon : public Striker`. При ударе — обычный урон + замораживание (выводит сообщение и добавляет 5 урона).

Оберни меч: `CriticalStrike(FrostWeapon(Weapon("Ice Sword", 20), 5), 20)` — ледяной меч с критом.

### 9.2. Интерфейс Fightable

Создай интерфейс `Fightable`:

```cpp
class Fightable {
public:
    virtual void takeDamage(int dmg) = 0;
    virtual bool isAlive() const = 0;
    virtual std::string getName() const = 0;
    virtual ~Fightable() {}
};
```

Пусть и `Player`, и `Monster` реализуют `Fightable`. Теперь `Player::attack` принимает `Fightable&` — и может бить любого: игрока, монстра, будущего NPC.

### 9.3. Арена

Создай 4 бойцов с разным оружием:
- Knight: меч + яд + крит
- Mage: посох (высокий урон, без обёрток)
- Rogue: кинжал + крит (высокий шанс)
- Archer: лук + лёд

Проведи круговой турнир: каждый против каждого. Победитель получает очко. В конце — таблица результатов. Весь бой работает через `Striker&` и `Fightable&` — код турнира не знает конкретные типы.
