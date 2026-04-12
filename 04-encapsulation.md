# 04. Инкапсуляция: геттеры, сеттеры и указатель this

В уроке 02 мы спрятали поля в `private` и добавили методы. Теперь — как правильно давать доступ к закрытым данным и зачем нужен указатель `this`.

## Зачем геттеры и сеттеры

Допустим, у персонажа есть здоровье. Мы спрятали его в `private` — правильно. Но теперь другой код не может даже узнать, сколько HP у персонажа. И не может изменить HP (например, при лечении).

Для этого пишут два типа методов:
- **геттер** (get) — возвращает значение поля
- **сеттер** (set) — устанавливает значение поля

```cpp
class Player {
private:
    std::string name;
    int hp;
    int maxHp;

public:
    Player(std::string n, int h) : name(n), hp(h), maxHp(h) {}

    // геттеры
    std::string getName() const { return name; }
    int getHp() const { return hp; }
    int getMaxHp() const { return maxHp; }

    // сеттер
    void setHp(int value) {
        if (value < 0) value = 0;
        if (value > maxHp) value = maxHp;
        hp = value;
    }
};
```

## const в геттерах

Обрати внимание на `const` после скобок:

```cpp
int getHp() const { return hp; }
```

Это значит: "этот метод не изменяет объект". Компилятор проверит — если внутри `const`-метода попытаться изменить поле, будет ошибка.

Правило: геттеры всегда `const`. Они только читают, не меняют.

## Валидация в сеттерах

Главная сила сеттеров — **валидация**. Вместо прямого доступа к полю код проходит через проверку:

```cpp
void setHp(int value) {
    if (value < 0) value = 0;       // не ниже нуля
    if (value > maxHp) value = maxHp; // не выше максимума
    hp = value;
}
```

Теперь невозможно сделать HP отрицательным или выше максимума — сеттер не даст. Именно в этом смысл инкапсуляции: данные защищены логикой.

## Не каждому полю нужен геттер и сеттер

Распространённая ошибка — автоматически писать get/set для каждого поля. Это превращает класс в структуру с лишним кодом. По сути, такой класс ничем не отличается от `struct` — данные открыты, просто через лишнюю обёртку.

Хороший объект — это не "мешок с данными", а **живая сущность с поведением**. Он сам знает, что делать со своим состоянием.

Сравни два подхода:

```cpp
// плохо: объект — мешок данных
player.setHp(player.getHp() - damage);

// хорошо: объект — живая сущность
player.takeDamage(damage);
```

В первом случае вся логика снаружи: кто-то достаёт HP, считает, записывает обратно. Объект — пассивный контейнер.

Во втором — объект сам решает, как обработать урон. Может учесть броню, сопротивление, неуязвимость. И никто снаружи не лезет в его внутренности.

В уроке 02 мы писали `birthday()` вместо `setAge()` — это правильный подход. Методы описывают действия из жизни объекта, а не технические операции над полями.

**Когда геттеры/сеттеры всё-таки нужны:**
- Геттер — когда внешнему коду нужно *прочитать* значение (вывести HP на экран, сравнить два объекта)
- Сеттер — когда нужно *задать* начальное значение или когда объект действительно является простым контейнером данных (как `DateClass` из методички)

В лабораторной работе от вас потребуют геттеры и сеттеры — это часть задания. Но помни: в реальных проектах чем меньше get/set, тем лучше дизайн. Если ты пишешь сеттер — подумай, может, вместо него нужен метод с осмысленным названием.

## Указатель this

Допустим, ты пишешь сеттер и хочешь назвать параметр так же, как поле — потому что это логично:

```cpp
void setName(std::string name) {
    name = name; // ???
}
```

Запускаешь — имя не меняется. Почему? Компилятор думает, что оба `name` — это параметр, и присваивает его самому себе. Поле класса не затрагивается.

Для таких случаев есть `this` — указатель на текущий объект. Он доступен внутри любого метода.

### Решение конфликта имён

```cpp
void setName(std::string name) {
    name = name; // кто кому присваивается?
}
```

Компилятор думает, что `name` — это параметр, и присваивает параметр самому себе. Поле не меняется.

`this` решает проблему:

```cpp
void setName(std::string name) {
    this->name = name; // this->name — поле, name — параметр
}
```

### Цепочки вызовов

Если сеттер возвращает ссылку на текущий объект, можно вызывать методы цепочкой:

```cpp
class Player {
private:
    std::string name;
    int hp;
    int armor;

public:
    Player() : name("Unknown"), hp(100), armor(0) {}

    Player& setName(std::string name) {
        this->name = name;
        return *this;
    }

    Player& setHp(int hp) {
        this->hp = hp;
        return *this;
    }

    Player& setArmor(int armor) {
        this->armor = armor;
        return *this;
    }

    void print() const {
        std::cout << name << ": " << hp << " HP, " << armor << " armor\n";
    }
};
```

Теперь:

```cpp
Player hero;
hero.setName("Knight").setHp(120).setArmor(50);
hero.print(); // Knight: 120 HP, 50 armor
```

`return *this` — возвращает текущий объект по ссылке. `this` — указатель, `*this` — сам объект.

## Полный пример

```cpp
#include <iostream>
#include <string>
#include <Windows.h>

class Player {
private:
    std::string name;
    int hp;
    int maxHp;

public:
    Player(std::string n, int h) : name(n), hp(h), maxHp(h) {}

    std::string getName() const { return name; }
    int getHp() const { return hp; }
    int getMaxHp() const { return maxHp; }

    void setHp(int value) {
        if (value < 0) value = 0;
        if (value > maxHp) value = maxHp;
        hp = value;
    }

    void takeDamage(int dmg) {
        setHp(hp - dmg);
    }

    void heal(int amount) {
        setHp(hp + amount);
    }

    bool isAlive() const {
        return hp > 0;
    }

    void printStatus() const {
        std::cout << name << ": " << hp << "/" << maxHp << " HP\n";
    }
};

int main() {
    SetConsoleCP(1251);
    SetConsoleOutputCP(1251);

    Player knight("Knight", 100);
    knight.printStatus();

    knight.takeDamage(70);
    knight.printStatus();

    knight.heal(50);
    knight.printStatus();

    knight.takeDamage(200);
    knight.printStatus();
    std::cout << "Жив: " << knight.isAlive() << "\n";

    return 0;
}
```

```text
Knight: 100/100 HP
Knight: 30/100 HP
Knight: 80/100 HP
Knight: 0/100 HP
Жив: 0
```

`takeDamage` и `heal` используют `setHp` внутри — валидация работает автоматически. HP не уходит ниже 0 и выше 100.

## Шпаргалка

- Геттер — возвращает значение private-поля: `int getHp() const`
- Сеттер — устанавливает значение с проверкой: `void setHp(int value)`
- `const` после метода — обещание не менять объект
- Не каждому полю нужен геттер/сеттер — только когда действительно нужен доступ
- `this` — указатель на текущий объект
- `this->field` — обращение к полю, когда имя параметра совпадает
- `return *this` — возврат объекта для цепочки вызовов

## Практика

### 4.1. Зелье с валидацией

Создай класс `Potion` с полями: название, сила действия (`power`), количество зарядов (`charges`).

Требования:
- `power` не может быть меньше 1 и больше 100
- `charges` не может быть меньше 0
- Метод `use()` уменьшает заряды на 1 и возвращает `true`. Если зарядов 0 — ничего не делает и возвращает `false`

Проверь:

```cpp
Potion heal("Heal", 30, 2);
std::cout << heal.use() << "\n"; // 1
std::cout << heal.use() << "\n"; // 1
std::cout << heal.use() << "\n"; // 0
std::cout << "Зарядов: " << heal.getCharges() << "\n"; // 0
```

### 4.2. Цепочка настройки

Создай класс `Monster` с полями: имя, здоровье, урон. Все сеттеры возвращают `Monster&` для цепочки.

Проверь:

```cpp
Monster goblin;
goblin.setName("Goblin").setHp(50).setDamage(10);
goblin.print(); // Goblin: 50 HP, урон 10
```

### 4.3. Бой с геттерами

Создай двух персонажей и проведи бой в цикле. После каждого удара выводи состояние обоих. Используй `getHp()` и `isAlive()` для проверки условий.

```cpp
Player knight("Knight", 100);
Player rogue("Rogue", 80);
int knightDmg = 25, rogueDmg = 35;

// бой: по очереди бьют друг друга, пока кто-то жив
```

Ожидаемый вывод (приблизительно):

```text
Knight бьёт Rogue на 25
Knight: 100/100 HP | Rogue: 55/80 HP

Rogue бьёт Knight на 35
Knight: 65/100 HP | Rogue: 55/80 HP

...
```
