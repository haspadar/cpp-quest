# 11. Объекты меняют поведение

В прошлых уроках мы строили иерархию: `Character` → `Player`, `Goblin`, `Dragon`. Каждый атакует по-своему. Но что если персонаж хочет **сменить стиль** атаки прямо в бою?

## Проблема: поведение зашито намертво

Воин всегда бьёт мечом. Но что если он поднял лук? Или перешёл в оборону? Или начал использовать магию?

```cpp
class Player : public Character {
    void attack(Fightable& target) {
        // всегда одно и то же
        int dmg = weapon.strike();
        target.takeDamage(dmg);
    }
};
```

Чтобы сменить стиль, нужно менять код метода. Или городить `if`:

```cpp
void attack(Fightable& target) {
    if (style == "melee") {
        // удар мечом
    } else if (style == "ranged") {
        // выстрел из лука
    } else if (style == "magic") {
        // заклинание
    } else if (style == "defensive") {
        // блок + слабый удар
    }
    // каждый новый стиль — новый if
}
```

Каждый новый стиль — новый `if`. Метод `attack` растёт и знает слишком много.

## Стиль атаки — это объект

А что если стиль атаки — отдельный объект со своим `execute`?

```cpp
class AttackStyle {
public:
    virtual void execute(const std::string& attackerName,
                        Fightable& target) = 0;
    virtual ~AttackStyle() {}
};
```

Конкретные стили:

```cpp
class MeleeAttack : public AttackStyle {
private:
    Striker& weapon;
public:
    MeleeAttack(Striker& w) : weapon(w) {}

    void execute(const std::string& name, Fightable& target) override {
        int dmg = weapon.strike();
        std::cout << name << " рубит " << target.getName()
                  << " на " << dmg << "!\n";
        target.takeDamage(dmg);
    }
};

class RangedAttack : public AttackStyle {
private:
    int damage;
    int arrows;
public:
    RangedAttack(int dmg, int arr) : damage(dmg), arrows(arr) {}

    void execute(const std::string& name, Fightable& target) override {
        if (arrows > 0) {
            arrows--;
            std::cout << name << " стреляет в " << target.getName()
                      << "! Стрел: " << arrows << "\n";
            target.takeDamage(damage);
        } else {
            std::cout << name << ": стрелы кончились!\n";
        }
    }
};

class MagicAttack : public AttackStyle {
private:
    int spellPower;
    int& mana;  // ссылка на ману персонажа
    int manaCost;
public:
    MagicAttack(int power, int& mana, int cost)
        : spellPower(power), mana(mana), manaCost(cost) {}

    void execute(const std::string& name, Fightable& target) override {
        if (mana >= manaCost) {
            mana -= manaCost;
            std::cout << name << " кастует Fireball на "
                      << target.getName() << "! Мана: " << mana << "\n";
            target.takeDamage(spellPower);
        } else {
            std::cout << name << ": недостаточно маны!\n";
        }
    }
};
```

Каждый стиль — маленький объект, 15-20 строк. Знает только своё дело.

## Персонаж использует стратегию

```cpp
class Player : public Character {
private:
    AttackStyle* style;

public:
    Player(std::string n, int h, AttackStyle* s)
        : Character(n, h), style(s) {}

    void setStyle(AttackStyle* newStyle) {
        style = newStyle;
    }

    void attack(Fightable& target) {
        style->execute(name, target);
    }
};
```

Одна строка в `attack`. Стиль решает всё. Смена стиля — одна строка:

```cpp
Weapon sword("Sword", 30);
MeleeAttack melee(sword);
RangedAttack ranged(20, 10);

Player knight("Knight", 100, &melee);

knight.attack(goblin);    // рубит мечом

knight.setStyle(&ranged);
knight.attack(goblin);    // стреляет из лука
```

## Это паттерн «Стратегия»

**Стратегия** — поведение объекта, вынесенное в отдельный объект. Можно подменить в любой момент.

Сравни с тем, что мы уже знаем:
- **Декоратор** (урок 08) — обёртка добавляет поведение вокруг вызова
- **Стратегия** — заменяет поведение целиком

Декоратор: `PoisonedWeapon` оборачивает `Weapon` и **дополняет** `strike()`.
Стратегия: `MeleeAttack` **заменяет** весь способ атаки.

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

class AttackStyle {
public:
    virtual void execute(const std::string& name, Fightable& target) = 0;
    virtual ~AttackStyle() {}
};

class MeleeAttack : public AttackStyle {
private:
    int damage;
public:
    MeleeAttack(int d) : damage(d) {}
    void execute(const std::string& name, Fightable& target) override {
        std::cout << name << " рубит " << target.getName()
                  << " на " << damage << "!\n";
        target.takeDamage(damage);
    }
};

class RangedAttack : public AttackStyle {
private:
    int damage;
    int arrows;
public:
    RangedAttack(int d, int a) : damage(d), arrows(a) {}
    void execute(const std::string& name, Fightable& target) override {
        if (arrows > 0) {
            arrows--;
            std::cout << name << " стреляет в " << target.getName()
                      << "! (" << arrows << " стрел)\n";
            target.takeDamage(damage);
        } else {
            std::cout << name << ": стрелы кончились!\n";
        }
    }
};

class DefensiveStance : public AttackStyle {
private:
    int weakDamage;
    int& targetHp;
    int healAmount;
public:
    DefensiveStance(int dmg, int& hp, int heal)
        : weakDamage(dmg), targetHp(hp), healAmount(heal) {}

    void execute(const std::string& name, Fightable& target) override {
        std::cout << name << " защищается и контратакует!\n";
        target.takeDamage(weakDamage);
        targetHp += healAmount;
        std::cout << "  +" << healAmount << " HP от блока\n";
    }
};

class Player : public Character {
private:
    AttackStyle* style;

public:
    Player(std::string n, int h, AttackStyle* s)
        : Character(n, h), style(s) {}

    void setStyle(AttackStyle* s) { style = s; }

    void attack(Fightable& target) {
        style->execute(name, target);
    }

    int& hpRef() { return hp; }
};

int main() {
    SetConsoleCP(1251);
    SetConsoleOutputCP(1251);
    srand(time(0));

    MeleeAttack melee(30);
    RangedAttack ranged(20, 5);

    Player knight("Knight", 120, &melee);

    DefensiveStance defense(10, knight.hpRef(), 15);

    Character troll("Troll", 150);

    std::cout << knight << " vs " << troll << "\n\n";

    int round = 1;
    while (knight.isAlive() && troll.isAlive()) {
        std::cout << "--- Ход " << round << " ---\n";

        // Тактика: если HP мало — защита, иначе — атака
        if (knight.hpRef() < 50) {
            knight.setStyle(&defense);
        } else if (round <= 5) {
            knight.setStyle(&ranged);  // сначала стрелы
        } else {
            knight.setStyle(&melee);   // потом меч
        }

        knight.attack(troll);

        if (troll.isAlive()) {
            std::cout << "Troll бьёт Knight!\n";
            knight.takeDamage(25);
        }

        std::cout << knight << " | " << troll << "\n\n";
        round++;
    }

    if (knight.isAlive()) {
        std::cout << "Knight победил!\n";
    } else {
        std::cout << "Troll победил...\n";
    }

    return 0;
}
```

Рыцарь меняет тактику по ситуации: стреляет, рубит мечом, уходит в оборону — и всё это без единого `if` внутри `attack()`.

## Шпаргалка

- Стратегия — поведение как отдельный объект, который можно подменить
- `AttackStyle*` — указатель на текущую стратегию
- `setStyle(newStyle)` — смена поведения в рантайме
- Декоратор дополняет поведение. Стратегия заменяет целиком.
- Каждый стиль — маленький объект (15-20 строк)

```cpp
class AttackStyle {
public:
    virtual void execute(const string& name, Fightable& target) = 0;
    virtual ~AttackStyle() {}
};

class Player {
    AttackStyle* style;
public:
    void setStyle(AttackStyle* s) { style = s; }
    void attack(Fightable& t) { style->execute(name, t); }
};
```

## Напоследок

1. Паттерн «Стратегия» — один из самых используемых в индустрии. В игровом AI стратегия определяет поведение монстра: «агрессивный», «осторожный», «убегающий». Монстр переключается между ними по ситуации.

2. В Java стратегии часто реализуют через лямбды: `button.setOnClick(() -> doSomething())`. В C++ тоже можно — через `std::function`, но мы пока используем интерфейсы, потому что они нагляднее.

3. В шахматных движках (Stockfish, Leela) стратегия оценки позиции — подменяемый объект. Это позволяет сравнивать разные алгоритмы оценки без изменения основного кода.

## Практика

### 11.1. Стиль защиты

Создай интерфейс `DefenseStyle` с методом `defend(int dmg)`, возвращающим реальный урон после защиты.

Реализуй:
- `NoDefense` — весь урон проходит
- `BlockDefense` — урон уменьшается на фиксированное число
- `DodgeDefense` — шанс полностью уклониться

Подключи к `Player`: `player.setDefense(&dodge)`.

### 11.2. AI монстра

Создай монстра, который меняет стратегию в зависимости от HP:
- Выше 50% HP — `AggressiveAttack` (высокий урон)
- Ниже 50% — `CautiousAttack` (средний урон + лечение себя)
- Ниже 20% — `DesperateAttack` (огромный урон, но ранит себя)

### 11.3. Арена со стратегиями

4 бойца, у каждого свой стиль атаки и защиты. Круговой турнир. Перед каждым боем бойцы «готовятся» — выбирают стратегию под противника (например, против тяжёлого бойца — магия, против быстрого — оборона).
