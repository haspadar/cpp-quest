# 08. Объекты оборачивают объекты

В прошлых уроках мы строили объекты из объектов: Player имеет Weapon, Weapon имеет урон. Теперь — другой способ комбинировать объекты: оборачивание.

## Когда хочется добавить эффект

У нас есть меч. Он наносит 25 урона. Всё работает.

Теперь хочется: меч с ядом. Он наносит 25 урона + 10 урона ядом. Как сделать?

Вариант 1 — добавить поле в `Weapon`:

```cpp
class Weapon {
private:
    std::string name;
    int damage;
    int poisonDamage = 0;  // новое поле
    bool hasCrit = false;   // и ещё одно
    int fireDamage = 0;     // и ещё...
};
```

Для каждого нового эффекта — новое поле. Огонь, лёд, крит, вампиризм — класс `Weapon` раздуется до 20 полей, большинство из которых будут нулями.

Вариант 2 — новый класс:

```cpp
class PoisonedSword {
    // копируем весь код Weapon...
    // добавляем яд...
};
```

Дублирование. А `PoisonedFireSword`? А `CriticalPoisonedFireSword`?

Оба варианта — тупик.

## Обёртка: добавляем поведение снаружи

Другой подход: не менять `Weapon`, а **обернуть** его:

```cpp
class PoisonedWeapon {
private:
    Weapon& base;
    int poisonDmg;

public:
    PoisonedWeapon(Weapon& w, int poison)
        : base(w), poisonDmg(poison) {}

    int strike() const {
        int baseDmg = base.strike();
        std::cout << "  Яд добавляет " << poisonDmg << " урона\n";
        return baseDmg + poisonDmg;
    }
};
```

`PoisonedWeapon` не копирует логику оружия. Он **держит ссылку** на настоящее оружие и добавляет к его урону яд.

```cpp
Weapon sword("Sword", 25);
PoisonedWeapon poisonedSword(sword, 10);

int dmg = poisonedSword.strike(); // 25 + 10 = 35
```

Меч не знает про яд. Яд не знает, как меч считает урон. Каждый делает своё.

## Ещё одна обёртка: крит

```cpp
class CriticalStrike {
private:
    Weapon& base;
    int critChance; // процент

public:
    CriticalStrike(Weapon& w, int chance)
        : base(w), critChance(chance) {}

    int strike() const {
        int dmg = base.strike();
        if (rand() % 100 < critChance) {
            std::cout << "  Критический удар!\n";
            dmg *= 2;
        }
        return dmg;
    }
};
```

```cpp
Weapon axe("Axe", 30);
CriticalStrike critAxe(axe, 25); // 25% шанс крита

int dmg = critAxe.strike(); // 30 или 60
```

## Это паттерн «Декоратор»

**Декоратор** — объект, который оборачивает другой объект и добавляет поведение, не меняя оригинал.

Как матрёшка: внутри — обычный меч. Снаружи — слой яда. Ещё снаружи — слой крита. Каждый слой добавляет своё, не зная о других.

Ключевая идея: оригинальный объект **не меняется**. `Weapon` не знает ни про яд, ни про крит. Эффекты — отдельные объекты.

## Проблема: разные типы

Сейчас есть неудобство. `PoisonedWeapon` и `CriticalStrike` — разные классы. Нельзя обернуть `PoisonedWeapon` в `CriticalStrike`, потому что `CriticalStrike` принимает `Weapon&`, а не `PoisonedWeapon&`.

Мы решим это в следующем уроке с помощью **интерфейсов** (`virtual`). Пока — работаем с одним уровнем обёртки. Этого достаточно для практики.

## Полный пример

```cpp
#include <iostream>
#include <string>
#include <cstdlib>
#include <ctime>
#include <Windows.h>

class Weapon {
private:
    std::string name;
    int damage;

public:
    Weapon() : name("Fist"), damage(5) {}
    Weapon(std::string n, int d) : name(n), damage(d) {}

    int strike() const {
        return damage;
    }

    std::string getName() const { return name; }

    friend std::ostream& operator<<(std::ostream& out, const Weapon& w) {
        out << w.name << " (урон: " << w.damage << ")";
        return out;
    }
};

class PoisonedWeapon {
private:
    Weapon& base;
    int poisonDmg;

public:
    PoisonedWeapon(Weapon& w, int poison)
        : base(w), poisonDmg(poison) {}

    int strike() const {
        int dmg = base.strike();
        std::cout << "  " << base.getName() << " отравляет цель! (+"
                  << poisonDmg << ")\n";
        return dmg + poisonDmg;
    }

    friend std::ostream& operator<<(std::ostream& out, const PoisonedWeapon& pw) {
        out << pw.base << " [яд: " << pw.poisonDmg << "]";
        return out;
    }
};

class CriticalStrike {
private:
    Weapon& base;
    int critChance;

public:
    CriticalStrike(Weapon& w, int chance)
        : base(w), critChance(chance) {}

    int strike() const {
        int dmg = base.strike();
        if (rand() % 100 < critChance) {
            std::cout << "  Критический удар!\n";
            dmg *= 2;
        }
        return dmg;
    }

    friend std::ostream& operator<<(std::ostream& out, const CriticalStrike& cs) {
        out << cs.base << " [крит: " << cs.critChance << "%]";
        return out;
    }
};

class VampiricWeapon {
private:
    Weapon& base;
    int healPercent;

public:
    VampiricWeapon(Weapon& w, int percent)
        : base(w), healPercent(percent) {}

    int strike() const {
        int dmg = base.strike();
        int heal = dmg * healPercent / 100;
        std::cout << "  Вампиризм: восстанавливает " << heal << " HP\n";
        return dmg;
    }

    int getHeal() const {
        return base.strike() * healPercent / 100;
    }

    friend std::ostream& operator<<(std::ostream& out, const VampiricWeapon& vw) {
        out << vw.base << " [вампиризм: " << vw.healPercent << "%]";
        return out;
    }
};

int main() {
    SetConsoleCP(1251);
    SetConsoleOutputCP(1251);
    srand(time(0));

    Weapon sword("Sword", 25);
    PoisonedWeapon poisonSword(sword, 10);
    CriticalStrike critSword(sword, 30);

    std::cout << "Обычный: " << sword << "\n";
    std::cout << "С ядом:  " << poisonSword << "\n";
    std::cout << "С критом: " << critSword << "\n\n";

    std::cout << "--- Обычный удар ---\n";
    int dmg1 = sword.strike();
    std::cout << "Урон: " << dmg1 << "\n\n";

    std::cout << "--- Отравленный удар ---\n";
    int dmg2 = poisonSword.strike();
    std::cout << "Урон: " << dmg2 << "\n\n";

    std::cout << "--- Удар с шансом крита ---\n";
    int dmg3 = critSword.strike();
    std::cout << "Урон: " << dmg3 << "\n\n";

    // Ещё один меч с вампиризмом
    Weapon axe("Axe", 40);
    VampiricWeapon vampAxe(axe, 25);

    std::cout << "Вампирский: " << vampAxe << "\n";
    std::cout << "--- Вампирский удар ---\n";
    int dmg4 = vampAxe.strike();
    std::cout << "Урон: " << dmg4 << "\n";

    return 0;
}
```

```text
Обычный: Sword (урон: 25)
С ядом:  Sword (урон: 25) [яд: 10]
С критом: Sword (урон: 25) [крит: 30%]

--- Обычный удар ---
Урон: 25

--- Отравленный удар ---
  Sword отравляет цель! (+10)
Урон: 35

--- Удар с шансом крита ---
  Критический удар!
Урон: 50

Вампирский: Axe (урон: 40) [вампиризм: 25%]
--- Вампирский удар ---
  Вампиризм: восстанавливает 10 HP
Урон: 40
```

Меч один, но три обёртки дают ему три разных поведения. И ни одна обёртка не знает о других.

## Шпаргалка

- Декоратор — объект, который оборачивает другой и добавляет поведение
- Оригинал не меняется — эффект снаружи, не внутри
- Обёртка хранит ссылку на оригинал: `Weapon& base`
- Обёртка вызывает метод оригинала и добавляет своё: `base.strike() + poisonDmg`
- Вместо 10 полей в одном классе — 10 маленьких обёрток
- Пока обёртки не комбинируются друг с другом — для этого нужны интерфейсы (следующий урок)

```cpp
class PoisonedWeapon {
    Weapon& base;
    int poisonDmg;
public:
    PoisonedWeapon(Weapon& w, int p) : base(w), poisonDmg(p) {}
    int strike() const { return base.strike() + poisonDmg; }
};
```

## Напоследок

1. Паттерн «Декоратор» описан в книге «Design Patterns» (1994) — её написали четверо авторов, которых называют «Банда четырёх» (Gang of Four). Книга до сих пор актуальна.

2. В Java декораторы повсюду: `BufferedReader(new FileReader("file.txt"))` — `BufferedReader` оборачивает `FileReader` и добавляет буферизацию. Тот же принцип, что и наш `PoisonedWeapon(sword)`.

3. В игре Space Invaders (проект Junior Objects Егора Бугаенко) пуля собирается из обёрток: `patched(div, kill, missed, outside, grave)`. Каждая обёртка — 20 строк. Обнаружение попадания, выход за экран, удаление — всё отдельные объекты. Ни один из них не знает о других.

## Практика

### 8.1. Огненное оружие

Создай обёртку `FireWeapon`, которая добавляет огненный урон. Огненный урон — фиксированное число, заданное при создании.

```cpp
Weapon bow("Bow", 20);
FireWeapon fireBow(bow, 15);
std::cout << fireBow << "\n";  // Bow (урон: 20) [огонь: 15]
int dmg = fireBow.strike();    // 35
```

### 8.2. Благословлённое оружие

Создай обёртку `BlessedWeapon`: каждый удар лечит владельца на фиксированное количество HP.

Метод `strike()` возвращает урон как обычно. Но при ударе нужно передать информацию о лечении. Подумай: как обёртка сообщит игроку, сколько HP восстановить? Варианты:
- Метод `getHeal()`, который возвращает количество лечения
- Метод `strike(Player& wielder)`, который сам лечит

Какой вариант лучше? Попробуй оба.

### 8.3. Бой с эффектами

Создай двух игроков:
- Knight с отравленным мечом (Sword + 10 яда)
- Rogue с вампирским кинжалом (Dagger + 30% вампиризм)

Проведи бой. Knight наносит урон + яд. Rogue наносит урон и лечится.

Подумай: как передать эффект обёртки в `Player::attack`? Игрок сейчас хранит `Weapon`, а не обёртку. Это неудобно — и именно эту проблему решит следующий урок.
