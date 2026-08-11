
# Brew & Distillation Controller

Универсальный встраиваемый контроллер технологических процессов на базе **STM32F407VETx** для автоматизации домашнего и лабораторного оборудования.

Основные режимы работы:

* **Пиво** — автоматизация нагрева, температурных пауз, циркуляции, кипячения и охлаждения;
* **Дистилляция** — управление нагревом, насосами, температурным контролем, технологическими стадиями и аварийной защитой;
* **Ручной режим** — ручное управление исполнительными устройствами в пределах разрешений Safety Manager;
* **Настройки и диагностика** — калибровка датчиков, PID, назначение каналов, тестирование оборудования и системная диагностика.

Проект разрабатывается как модульная embedded-система с **Clean Architecture**, аппаратной абстракцией, автоматизированным тестированием и CI/CD.

---

# 1. Статус проекта

Текущий статус:

**Early Development / GT-00 — Engineering Environment**

Текущая baseline-версия:

`0.0.1`

Текущий этап:

`GT-00 — Repository & Engineering Environment`

Текущая задача Developer A:

`GT00-A001 — STM32F407VE baseline project`

До версии `1.0.0` проект не следует считать завершенным промышленным контроллером.

---

# 2. Цель проекта

Создать автономный контроллер, способный управлять несколькими типами тепловых технологических процессов через единое аппаратное и программное ядро.

Вместо создания отдельных прошивок:

```text
beer_firmware
distillation_firmware
```

используется одна система:

```text
                 Process Controller
                         │
             ┌───────────┴───────────┐
             │                       │
             ▼                       ▼
          Beer App             Distillation App
             │                       │
             └───────────┬───────────┘
                         │
                         ▼
                  Process Engine
                         │
           ┌─────────────┼─────────────┐
           ▼             ▼             ▼
      Temperature      Heater         Pump
           │             │             │
           └─────────────┼─────────────┘
                         ▼
                   Safety Manager
                         │
                         ▼
                    STM32F407VE
```

Общими для всех технологических режимов являются:

* датчики температуры;
* фильтрация измерений;
* PID;
* управление нагревателем;
* управление насосами;
* таймеры;
* журналирование;
* хранение конфигурации;
* аварийная защита;
* пользовательский интерфейс;
* диагностика;
* восстановление после перезапуска.

---

# 3. Основной контроллер

Используется:

**STM32F407VETx**

Основные характеристики целевой конфигурации:

* ARM Cortex-M4;
* аппаратный FPU;
* максимальная частота CPU — 168 MHz;
* корпус LQFP100;
* 512 KB Flash;
* аппаратные таймеры;
* ADC;
* DMA;
* FSMC;
* SPI;
* UART;
* I²C;
* watchdog;
* CRC.

Целевая рабочая частота:

```text
SYSCLK = 168 MHz

AHB  = 168 MHz
APB1 = 42 MHz
APB2 = 84 MHz
```

Окончательная настройка тактирования выполняется в `GT00-A002`.

---

# 4. Среда разработки

Проект фиксирует конкретные версии основных инструментов.

## IDE

```text
STM32CubeIDE 2.1.1
```

## MCU configurator

```text
STM32CubeMX 6.x
```

Файл CubeMX:

```text
firmware/brew_controller.ioc
```

должен храниться в Git.

## STM32 Firmware Package

```text
STM32CubeF4 1.28.3
```

## GUI

```text
STemWin
```

Используется версия STemWin из соответствующего пакета STM32CubeF4.

## Compiler

Основной ARM toolchain:

```text
arm-none-eabi-gcc
```

Host tests компилируются desktop-компилятором через CMake.

---

# 5. Дисплей

Архитектура предусматривает несколько TFT-контроллеров.

Основные варианты:

```text
SSD1289
ILI9341
SSD1963
```

Основной GUI framework:

```text
STemWin
```

Application и Domain не должны зависеть от конкретного контроллера TFT.

Архитектура:

```text
STemWin
   │
   ▼
Display Abstraction
   │
   ├── SSD1289 Driver
   ├── ILI9341 Driver
   └── SSD1963 Driver
```

Основным перспективным дисплеем считается SSD1963 с разрешением до 800×480.

---

# 6. Начальное меню

После загрузки и успешного POST пользователь получает главное меню:

```text
┌──────────────────────────────┐
│     PROCESS CONTROLLER       │
│                              │
│          ПИВО                │
│                              │
│       ДИСТИЛЛЯЦИЯ            │
│                              │
│       РУЧНОЙ РЕЖИМ           │
│                              │
│       НАСТРОЙКИ              │
│                              │
└──────────────────────────────┘
```

Выбор режима не загружает отдельную прошивку.

Запускается соответствующее приложение:

```text
BeerApplication
```

или:

```text
DistillationApplication
```

поверх общего Process Engine.

---

# 7. Температурные датчики

Используются:

**5 × NTC 10K**

Вместо DS18B20.

Предполагаемые характеристики:

```text
R25 ≈ 10 kΩ
B ≈ 3950 K
```

Рекомендуется использовать опорные резисторы:

```text
10 kΩ ±0.1%
```

Измерение:

```text
NTC
 │
 ▼
ADC
 │
 ▼
DMA
 │
 ▼
Digital filtering
 │
 ▼
Resistance calculation
 │
 ▼
Temperature calculation
 │
 ▼
Calibration
 │
 ▼
Temperature Service
```

Температура может рассчитываться:

* Beta equation;
* Steinhart-Hart.

Каждый из пяти каналов имеет собственную калибровку.

---

# 8. Назначение датчиков

Датчики физически обозначаются:

```text
TEMP1
TEMP2
TEMP3
TEMP4
TEMP5
```

Их технологическая роль задается конфигурацией.

Пример для пивоварения:

```text
TEMP1 → Mash
TEMP2 → Wort
TEMP3 → Hot Water
TEMP4 → Cooling
TEMP5 → Auxiliary
```

Пример для другого технологического процесса:

```text
TEMP1 → Vessel
TEMP2 → Process Sensor 1
TEMP3 → Process Sensor 2
TEMP4 → Cooling
TEMP5 → Auxiliary
```

Роли не должны быть жестко зашиты в ADC Driver.

---

# 9. Управление ТЭНом

Основной вариант:

```text
SSR-40DA
```

Также архитектура предусматривает возможность использования отдельного силового симисторного модуля.

Управление AC ТЭНом производится преимущественно через:

```text
Time Proportional Control
```

Например, при окне управления 2 секунды и выходе PID 60%:

```text
SSR ON  = 1.2 s
SSR OFF = 0.8 s
```

Высокочастотный PWM GPIO для обычного zero-cross AC SSR не используется как основной способ регулирования.

---

# 10. Управление насосом

Поддерживаются различные силовые варианты:

```text
SSR-10DA
D3808HK
SSR-10DD
```

Верхнеуровневая логика не знает, каким силовым модулем управляется насос.

Используется абстракция:

```cpp
class IPump
{
public:
    virtual ~IPump() = default;

    virtual void start() = 0;
    virtual void stop() = 0;
    virtual void setPower(uint8_t percent) = 0;
};
```

Backend может быть заменен без изменения BeerApplication или Process Engine.

---

# 11. Безопасность

Контроллер взаимодействует с:

* сетевым напряжением;
* мощными нагревателями;
* горячими жидкостями;
* насосами;
* потенциально пожароопасными технологическими процессами.

Программное управление не является заменой аппаратной защиты.

Проект должен предусматривать независимые аппаратные средства:

* автоматический выключатель;
* УЗО;
* защитное заземление;
* аппаратное аварийное отключение;
* независимый термостат/термопредохранитель;
* контактор отключения нагревателя;
* безопасное состояние исполнительных устройств при Reset;
* соответствующие номиналу проводники, разъемы и защитные компоненты.

Силовая часть должна быть гальванически и конструктивно отделена от логической части STM32.

Эксплуатация оборудования должна соответствовать действующим местным требованиям и законодательству.

---

# 12. Safety Manager

Все команды исполнительным устройствам проходят через Safety Manager.

Запрещена архитектура:

```text
GUI
 ↓
GPIO
 ↓
SSR
```

Правильная цепочка:

```text
GUI
 ↓
Application
 ↓
Process Engine
 ↓
Control Service
 ↓
Safety Manager
 ↓
Output Driver
 ↓
BSP
 ↓
HAL
```

Safety Manager имеет право отменить любую команду Process Engine.

---

# 13. Основные аварии

Архитектура предусматривает как минимум:

```text
SENSOR_OPEN
SENSOR_SHORT
SENSOR_OUT_OF_RANGE

OVER_TEMPERATURE

HEATER_STUCK_ON

HEATER_CONTROL_FAILURE

PUMP_FAILURE

COOLING_FAILURE

PROCESS_TIMEOUT

INVALID_CONFIGURATION

WATCHDOG_RESET

BROWNOUT

STORAGE_FAILURE
```

Уровни событий:

```text
INFO
WARNING
ERROR
EMERGENCY
```

При `EMERGENCY` система должна переходить в заранее определенное безопасное состояние.

---

# 14. Clean Architecture

Проект строго разделяется на слои.

```text
                 ┌─────────────────┐
                 │       GUI       │
                 │     STemWin     │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │   Application   │
                 │                 │
                 │ Beer / Distill  │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │     Domain      │
                 │                 │
                 │ Process         │
                 │ Recipe          │
                 │ PID             │
                 │ Alarm           │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │    Services     │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │     Drivers     │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │       BSP       │
                 └────────┬────────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │    STM32 HAL    │
                 └─────────────────┘
```

---

# 15. Главное правило архитектуры

Domain не должен знать о STM32.

Запрещено:

```text
Domain → stm32f4xx_hal.h
Domain → HAL_GPIO_WritePin()
Domain → STemWin
Domain → SSD1963
```

Также запрещено:

```text
GUI → HAL
GUI → GPIO
GUI → SSR Driver
BeerProcess → GPIO
```

Domain и большая часть Application должны собираться на обычном PC.

---

# 16. Структура репозитория

Целевая структура:

```text
brew-controller/
│
├── .github/
│   ├── workflows/
│   │   ├── build.yml
│   │   ├── tests.yml
│   │   └── release.yml
│   │
│   └── CODEOWNERS
│
├── ci/
│   ├── scripts/
│   │   ├── build.sh
│   │   ├── test.sh
│   │   ├── static_analysis.sh
│   │   └── package.sh
│   │
│   └── docker/
│
├── docs/
│   ├── architecture/
│   ├── development/
│   ├── hardware/
│   └── testing/
│
├── firmware/
│   │
│   ├── Core/
│   │   ├── Inc/
│   │   └── Src/
│   │
│   ├── Drivers/
│   │   ├── CMSIS/
│   │   └── STM32F4xx_HAL_Driver/
│   │
│   ├── BSP/
│   │
│   ├── Platform/
│   │
│   ├── Services/
│   │
│   ├── Domain/
│   │
│   ├── App/
│   │
│   ├── GUI/
│   │
│   ├── Middlewares/
│   │
│   ├── brew_controller.ioc
│   ├── startup_stm32f407xx.s
│   └── STM32F407VETX_FLASH.ld
│
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── mocks/
│   ├── fakes/
│   └── fixtures/
│
├── tools/
│
├── scripts/
│   ├── build_local.bat
│   ├── test_local.bat
│   └── clean.bat
│
├── artifacts/
│
├── .clang-format
├── .clang-tidy
├── .editorconfig
├── .gitignore
├── CMakeLists.txt
├── README.md
├── CONTRIBUTING.md
├── CHANGELOG.md
└── LICENSE
```

---

# 17. Ответственность Developer A

Developer A отвечает преимущественно за:

```text
firmware/BSP/
firmware/Platform/
firmware/Drivers/
firmware/Services/
```

Основные области:

* STM32CubeMX;
* Clock Tree;
* GPIO;
* ADC;
* DMA;
* timers;
* NTC;
* SSR;
* Pump;
* PID backend;
* watchdog;
* CRC;
* storage;
* hardware safety;
* hardware diagnostics;
* boot/recovery.

---

# 18. Ответственность Developer B

Developer B отвечает преимущественно за:

```text
firmware/App/
firmware/Domain/
firmware/GUI/
```

Основные области:

* Clean Architecture;
* Process Engine;
* Beer Application;
* технологические профили;
* Recipe Engine;
* STemWin;
* Screen Manager;
* Navigation;
* System State;
* UI State;
* Command/Event architecture.

---

# 19. Ответственность QA

QA отвечает преимущественно за:

```text
tests/
ci/
docs/testing/
```

Основные задачи:

* Unit Tests;
* Component Tests;
* Integration Tests;
* hardware tests;
* HIL;
* regression;
* negative tests;
* CI validation;
* release verification;
* test reports.

Разработчики также обязаны создавать unit tests для собственного кода.

---

# 20. Процесс разработки

Стандартный поток:

```text
Issue
  ↓
Feature Branch
  ↓
Implementation
  ↓
Unit Tests
  ↓
Local Build
  ↓
Push
  ↓
CI
  ↓
Pull Request
  ↓
Code Review
  ↓
QA
  ↓
Merge
```

---

# 21. Git Branches

Используются:

```text
main
develop
feature/*
bugfix/*
release/*
```

## main

Содержит только проверенные версии.

Прямой push запрещен.

## develop

Главная интеграционная ветка разработки.

## feature

Пример:

```text
feature/gt00-a001-baseline
feature/gt02-stemwin
feature/gt13-beer-engine
```

## bugfix

Пример:

```text
bugfix/ntc-open-detection
```

## release

Пример:

```text
release/1.0.0
```

---

# 22. Commit Style

Используются Conventional Commits.

Примеры:

```text
feat(platform): create STM32F407VE baseline project

feat(ntc): add five channel temperature driver

feat(gui): add main process selection screen

feat(beer): add mash process state machine

fix(pid): prevent integral windup

fix(safety): disable heater on sensor failure

test(ntc): add temperature conversion tests

ci(build): add firmware build pipeline

docs: update project architecture
```

---

# 23. Pull Requests

Каждая значимая задача должна проходить Pull Request.

PR должен содержать:

```text
Ticket

Description

Changes

Tests

Hardware verification

Known risks

Documentation changes
```

Перед merge обязательны:

```text
Build              PASS
Unit Tests         PASS
Static Analysis    PASS
Code Review        PASS
Required QA        PASS
```

---

# 24. Coding Style

Основной стиль:

```text
Classes        PascalCase
Interfaces     IPascalCase
Functions      camelCase
Variables      camelCase
Constants      kConstantName
Enums          PascalCase
```

Пример:

```cpp
class ITemperatureSensor
{
public:
    virtual ~ITemperatureSensor() = default;

    virtual float getTemperature() const = 0;
};
```

Форматирование выполняется через:

```text
clang-format
```

---

# 25. Generated Code

Код, автоматически создаваемый STM32CubeMX, должен быть максимально отделен от собственного кода.

CubeMX:

```text
Core/
Drivers/
```

Собственный код:

```text
BSP/
Platform/
Services/
Domain/
App/
GUI/
```

Не следует помещать бизнес-логику непосредственно в:

```text
main.c
stm32f4xx_it.c
stm32f4xx_hal_msp.c
```

---

# 26. USER CODE sections

Если изменение generated-файла необходимо, собственный код размещается только в участках:

```c
/* USER CODE BEGIN */

/* USER CODE END */
```

Но даже эти зоны не следует использовать для больших модулей.

Например допустимо:

```c
/* USER CODE BEGIN 2 */

AppBootstrap_Init();

/* USER CODE END 2 */
```

Недопустимо размещать внутри `main.c` тысячи строк Beer Engine или GUI.

---

# 27. Application entry point

Целевая структура запуска:

```text
Reset
 ↓
startup
 ↓
HAL_Init()
 ↓
SystemClock_Config()
 ↓
CubeMX peripheral initialization
 ↓
Board initialization
 ↓
Platform initialization
 ↓
AppBootstrap
 ↓
Application
```

В конечном состоянии `main.c` должен оставаться небольшим интеграционным файлом.

---

# 28. Process Engine

Пивоварение и другие технологические режимы строятся поверх общего Process Engine.

Концепция:

```text
Process
 ├── Stage
 ├── Conditions
 ├── Actions
 ├── Transitions
 └── Safety Constraints
```

Пример Beer FSM:

```text
IDLE
 ↓
HEAT_WATER
 ↓
MASH_IN
 ↓
REST
 ↓
REST
 ↓
MASH_OUT
 ↓
TRANSFER
 ↓
BOIL
 ↓
COOLING
 ↓
FINISHED
```

Конкретный рецепт задает параметры стадий.

---

# 29. Recipe Engine

Рецепт не должен быть жестко закодирован в FSM.

Предполагаемая структура:

```text
Recipe
 ├── Metadata
 ├── Stage 1
 ├── Stage 2
 ├── Stage 3
 └── ...
```

Каждая стадия может содержать:

```text
Target temperature
Duration
Heater limit
Pump mode
Pump power
Transition condition
User notification
```

---

# 30. PID Controller

PID является независимым Domain/Control компонентом.

Минимальные возможности:

* P;
* I;
* D;
* output limiting;
* integral limiting;
* anti-windup;
* configurable sample time;
* derivative filtering;
* manual/automatic mode;
* bumpless transfer.

PID не должен непосредственно работать с GPIO или SSR.

Правильно:

```text
Temperature
     │
     ▼
PID Controller
     │
     ▼
Power %
     │
     ▼
Heater Service
     │
     ▼
Safety Manager
     │
     ▼
SSR Driver
```

---

# 31. RTOS

Архитектура предусматривает использование FreeRTOS.

Предварительный набор задач:

```text
SensorTask
ControlTask
ProcessTask
GuiTask
AlarmTask
StorageTask
LoggerTask
```

Примерная частота:

```text
SensorTask    ~20 Hz
ControlTask   ~10 Hz
ProcessTask   ~5 Hz
GuiTask       ~10–20 Hz
AlarmTask     ~20 Hz
StorageTask   event-driven
LoggerTask    event-driven
```

Точные частоты определяются последующими GT и измерениями.

---

# 32. GUI Architecture

STemWin не должен содержать технологическую бизнес-логику.

Правильно:

```text
Button
 ↓
GUI Controller
 ↓
Application Command
 ↓
Application
 ↓
Domain
```

Например:

```text
Start Beer
 ↓
StartBeerCommand
 ↓
BeerApplication
 ↓
ProcessEngine.start()
```

---

# 33. System State

GUI получает данные через модель состояния системы.

Например:

```text
SystemState
ProcessState
TemperatureState
HeaterState
PumpState
AlarmState
```

GUI не должен самостоятельно читать ADC.

---

# 34. Настройки

Планируется хранить:

* калибровку пяти NTC;
* назначение температурных каналов;
* PID;
* ограничения мощности;
* настройки насосов;
* пользовательские настройки;
* аварийные пороги;
* рецепты;
* состояние процесса;
* версию структуры конфигурации.

Каждый persistent object должен иметь:

```text
magic
version
size
CRC
```

---

# 35. CI/CD

Каждый push/PR должен запускать автоматические проверки.

Pipeline:

```text
Commit / PR
     │
     ├── Format Check
     │
     ├── Static Analysis
     │
     ├── Host Build
     │
     ├── Unit Tests
     │
     ├── Firmware Build
     │
     ├── Firmware Size
     │
     └── Artifacts
```

---

# 36. Format Check

Используется:

```text
clang-format
```

CI должен обнаруживать файлы, не соответствующие принятому форматированию.

---

# 37. Static Analysis

Планируется использование:

```text
cppcheck
clang-tidy
```

Основной анализ проводится для собственного кода.

ST HAL, CMSIS и STemWin анализируются отдельно, чтобы внешние библиотеки не создавали большого количества нерелевантных предупреждений.

---

# 38. Host Build

Domain и часть Application должны собираться на обычном PC:

```text
cmake -S . -B build

cmake --build build
```

Это позволяет тестировать алгоритмы без STM32.

---

# 39. Unit Tests

Запуск:

```text
ctest --test-dir build --output-on-failure
```

На PC планируется тестировать:

* NTC mathematics;
* digital filters;
* PID;
* Process Engine;
* Beer FSM;
* Recipe Engine;
* Alarm logic;
* Safety rules;
* configuration validation;
* state machines;
* timers.

---

# 40. Firmware Build

Embedded build должен создавать:

```text
brew_controller.elf
brew_controller.hex
brew_controller.bin
brew_controller.map
```

---

# 41. Build Artifacts

Артефакты CI размещаются в:

```text
artifacts/
```

Релизный пакет:

```text
brew-controller-x.y.z/
├── brew_controller.bin
├── brew_controller.hex
├── brew_controller.elf
├── brew_controller.map
└── build-info.txt
```

---

# 42. build-info.txt

Пример:

```text
Project:
Brew & Distillation Controller

MCU:
STM32F407VETx

Firmware:
0.0.1

Git Commit:
abcdef12

STM32CubeIDE:
2.1.1

STM32CubeF4:
1.28.3

Build:
Debug
```

---

# 43. Firmware Versioning

Используется Semantic Versioning:

```text
MAJOR.MINOR.PATCH
```

Например:

```text
0.0.1
0.1.0
0.5.0
1.0.0
```

До `1.0.0` API и persistent data structures могут изменяться.

---

# 44. Milestones

Предварительная схема релизов:

```text
0.0.1
Infrastructure baseline

0.1.0
STM32 BSP

0.2.0
Temperature subsystem

0.3.0
Heater and pump

0.4.0
PID

0.5.0
Main GUI

0.6.0
Beer MVP

0.7.0
Beer recipes

0.8.0
Additional process mode MVP

0.9.0
Safety + Storage + Recovery

1.0.0
First complete release
```

---

# 45. Global Tickets

Проект реализуется последовательностью законченных Global Tickets.

```text
GT-00 Repository & Engineering Environment

GT-01 STM32F407VE BSP

GT-02 STemWin Minimal Port

GT-03 ADC + DMA

GT-04 NTC Driver

GT-05 Temperature Service

GT-06 GUI Framework

GT-07 Main Menu

GT-08 Heater Driver

GT-09 Pump Driver

GT-10 PID Controller

GT-11 Safety Manager

GT-12 Common Process Engine

GT-13 Beer MVP

GT-14 Beer Recipe Engine

GT-15 Beer GUI

GT-16 Additional Process MVP

GT-17 Additional Process GUI

GT-18 Configuration Manager

GT-19 Persistent Storage

GT-20 Recipe Editor

GT-21 Production Safety + Recovery

GT-22 Release 1.0
```

Каждый GT должен представлять собой отдельный проверяемый mini-product.

---

# 46. Current Ticket — GT-00

Цель GT-00:

получить полностью воспроизводимое Engineering Environment.

После:

```text
git clone
```

разработчик должен иметь возможность выполнить:

```text
configure
build
test
package
```

без ручного восстановления неизвестных зависимостей.

---

# 47. GT00-A001

Текущая задача Developer A:

```text
GT00-A001
Create STM32F407VE baseline project
```

На данном этапе включаются только минимальные возможности MCU.

Не настраиваются:

```text
ADC
DMA
NTC
SSR
Pump
STemWin
Process Engine
PID
FreeRTOS
```

Главный критерий:

```text
Clean
 ↓
Build
 ↓
0 Errors
 ↓
ELF generated
```

---

# 48. Testing Strategy

Уровни тестирования:

```text
L0 Static Checks

L1 Unit Tests

L2 Component Tests

L3 Integration Tests

L4 Hardware Tests

L5 HIL Tests

L6 System Tests
```

---

# 49. Hardware-In-The-Loop

В дальнейшем CI должен поддерживать тестовый STM32-стенд:

```text
CI Runner
   │
   ▼
ST-Link
   │
   ▼
STM32F407VE
   │
   ├── UART
   ├── simulated NTC
   ├── output sensing
   └── fault injection
```

HIL сможет автоматически:

* прошивать STM32;
* выполнять reset;
* анализировать UART;
* имитировать температуры;
* имитировать обрыв NTC;
* имитировать короткое замыкание NTC;
* проверять SSR output;
* проверять аварийное отключение.

---

# 50. Definition of Done

Общий DoD для задачи:

```text
[ ] Functionality implemented

[ ] Architecture rules respected

[ ] Project builds

[ ] Unit tests added

[ ] Unit tests pass

[ ] Static analysis passes

[ ] CI passes

[ ] Code review completed

[ ] Documentation updated

[ ] Hardware test completed when applicable

[ ] QA test passed

[ ] No unresolved Critical defects

[ ] No unresolved High defects
```

---

# 51. Local Development

Рекомендуемый рабочий каталог:

```text
D:\Projects\brew-controller
```

Рекомендуемый CubeIDE workspace:

```text
D:\STM32Workspace\BrewController
```

Workspace не следует помещать внутрь Git repository.

---

# 52. Clone

```bash
git clone <repository-url>
cd brew-controller
```

---

# 53. Open STM32 project

CubeMX project:

```text
firmware/brew_controller.ioc
```

Открывается через STM32CubeMX.

Firmware project открывается/импортируется в:

```text
STM32CubeIDE 2.1.1
```

---

# 54. STM32CubeMX regeneration

Перед генерацией убедиться, что используется правильный MCU:

```text
STM32F407VETx
```

и правильная версия firmware package:

```text
STM32CubeF4 1.28.3
```

После чего:

```text
GENERATE CODE
```

После каждой regeneration обязательно:

```text
git diff
```

Не следует автоматически commit'ить массовые изменения без понимания причины.

---

# 55. Clean Build

В STM32CubeIDE:

```text
Project
→ Clean
```

затем:

```text
Project
→ Build Project
```

или:

```text
Ctrl+B
```

---

# 56. Host Tests

После появления host-test инфраструктуры:

```bash
cmake -S . -B build
cmake --build build
ctest --test-dir build --output-on-failure
```

---

# 57. Repository Hygiene

В Git не должны попадать:

```text
Debug/
Release/
build/
.metadata/
*.o
*.d
*.su
*.log
```

Но должны храниться необходимые для воспроизводимости:

```text
*.ioc
.project
.cproject
.settings/
*.ld
startup*.s
source files
headers
CMakeLists.txt
CI configuration
```

---

# 58. Documentation

Документация находится в:

```text
docs/
```

## Architecture

```text
docs/architecture/
```

## Hardware

```text
docs/hardware/
```

## Development

```text
docs/development/
```

## Testing

```text
docs/testing/
```

Для каждого крупного архитектурного решения рекомендуется создавать ADR:

```text
Architecture Decision Record
```

---

# 59. Recommended Documentation Files

Планируется создание:

```text
docs/architecture/architecture.md
docs/architecture/layers.md
docs/architecture/dependencies.md

docs/development/toolchain.md
docs/development/coding-standard.md
docs/development/git-workflow.md

docs/hardware/board.md
docs/hardware/memory-map.md
docs/hardware/power.md
docs/hardware/io-map.md

docs/testing/test-strategy.md
docs/testing/hil.md
```

---

# 60. Hardware Revision

Аппаратная конфигурация также должна версионироваться.

Например:

```text
HW Rev A
HW Rev B
HW Rev C
```

Firmware должна иметь возможность определить или конфигурировать поддерживаемую аппаратную ревизию.

---

# 61. Planned Hardware

Базовый состав:

```text
STM32F407VETx

5 × NTC 10K

TFT
 ├── SSD1289
 ├── ILI9341
 └── SSD1963

Heater SSR
 └── SSR-40DA

Pump control
 ├── SSR-10DA
 ├── SSR-10DD
 └── compatible regulator/driver

Encoder

Buttons

Buzzer

Status outputs

Nonvolatile storage
```

Состав может изменяться по мере прохождения Hardware GT.

---

# 62. Future Extensions

Архитектура должна позволять дальнейшее добавление:

* SD card;
* USB Mass Storage;
* USB service interface;
* RTC;
* FRAM;
* Wi-Fi;
* ESP32;
* Ethernet;
* Flutter application;
* WebSocket;
* JSON protocol;
* recipe import/export;
* process history;
* temperature graphs;
* remote monitoring;
* automatic PID tuning;
* power-loss recovery;
* additional pumps;
* additional heating zones;
* flow sensors;
* liquid-level sensors;
* independent safety sensors.

Эти функции не входят автоматически в Release 1.0.

---

# 63. Non-Goals for Early Development

На ранних GT не ставятся задачи:

* создать сразу законченный внешний дизайн;
* реализовать облачный сервис;
* реализовать мобильное приложение;
* реализовать AI;
* реализовать все рецепты;
* поддерживать все возможные TFT;
* оптимизировать каждый байт RAM;
* преждевременно переводить весь HAL на LL.

Приоритет:

```text
Correctness
   ↓
Testability
   ↓
Safety
   ↓
Maintainability
   ↓
Performance optimization
```

---

# 64. Project Principles

## 1. Safety first

Ошибочное состояние должно приводить к безопасному поведению.

## 2. No direct hardware access from Domain

Domain остается аппаратно-независимым.

## 3. Every stage is testable

Каждый GT должен заканчиваться проверяемым mini-product.

## 4. No giant main.c

Бизнес-логика находится в отдельных модулях.

## 5. Configuration over hardcoding

Роли датчиков, PID, ограничения и рецепты должны конфигурироваться.

## 6. Tests are part of implementation

Задача без предусмотренных тестов не считается полностью завершенной.

## 7. Reproducible builds

Версии инструментов и библиотек фиксируются.

## 8. Generated code is isolated

CubeMX-код не становится местом хранения Domain/Application logic.

## 9. CI must be trusted

CI специально тестируется на корректное обнаружение ошибок.

## 10. Documentation evolves with code

Архитектурные изменения должны отражаться в документации.

---

# 65. Team

Проект рассчитан минимум на три параллельных роли:

```text
Developer A
Platform / Hardware / Control

Developer B
Application / Domain / GUI

QA
Testing / CI / HIL / Regression
```

---

# 66. License

Тип лицензии должен быть окончательно определен до публичного Release 1.0.

До выбора лицензии содержимое репозитория не следует автоматически считать разрешенным для неограниченного коммерческого использования или распространения.

---

# 67. Disclaimer

Этот проект является инженерной системой управления оборудованием и может взаимодействовать с потенциально опасной силовой частью.

Авторы программного обеспечения не должны рассматривать программные проверки как замену:

* электротехническим средствам защиты;
* аппаратным термостатам;
* аварийному отключению;
* безопасной конструкции оборудования;
* соблюдению местных норм и законодательства.

Перед эксплуатацией силовой установки должны быть отдельно проверены электрическая, тепловая, пожарная и технологическая безопасность.

---

# 68. Current Development Goal

Текущая ближайшая цель:

```text
GT-00
Repository & Engineering Environment
```

После завершения GT-00 команда должна получить:

```text
Fresh Machine
     ↓
git clone
     ↓
configure
     ↓
build
     ↓
unit tests
     ↓
static analysis
     ↓
firmware artifacts
     ↓
flash
     ↓
SYSTEM READY
```

После этого начинается последовательная реализация аппаратной платформы, GUI и технологических приложений.

---

**Project:** Brew & Distillation Controller
**MCU:** STM32F407VETx
**IDE:** STM32CubeIDE 2.1.1
**STM32 Firmware:** STM32CubeF4 1.28.3
**GUI:** STemWin
**Architecture:** Clean Architecture
**Current Release:** 0.0.1
**Current Milestone:** GT-00



//////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
Brew Controller — GT-00 / A-001

Базовый STM32CubeIDE-проект для STM32F407VETx

Этот каталог содержит базовый firmware-проект универсального контроллера пивоварения и технологических тепловых процессов на базе STM32F407VETx.

Текущий этап:

GT-00 — Repository & Engineering EnvironmentA-001 — Создать базовый STM32CubeIDE-проект

На этапе A-001 создается только минимальная аппаратно-программная основа проекта. Функциональность пивоварения, технологических профилей, NTC-датчиков, ТЭНа, насосов, дисплея и PID-регуляторов здесь намеренно отсутствует.

1. Цель этапа

Цель A-001 — получить минимальный, чистый и воспроизводимый проект STM32F407VETx, который:

открывается и собирается в STM32CubeIDE;

имеет сохраненный CubeMX-файл .ioc;

использует зафиксированную версию STM32CubeF4;

содержит корректные startup- и linker-файлы;

не содержит преждевременно подключенной периферии;

пригоден для хранения в Git;

может быть собран другим разработчиком или тестировщиком после чистого клонирования репозитория.

После выполнения этапа должна успешно выполняться цепочка:

Clean repository
      ↓
Open project
      ↓
Clean
      ↓
Build
      ↓
0 compilation errors
      ↓
firmware ELF generated

2. Аппаратная платформа

Микроконтроллер

STM32F407VETx

Основные характеристики используемого варианта:

ядро ARM Cortex-M4;

аппаратный FPU;

корпус LQFP100;

512 KB Flash;

семейство STM32F4.

При создании проекта необходимо выбрать именно:

STM32F407VETx

Не следует заменять его на:

STM32F407VGT
STM32F407ZGT
STM32F407VCT

без отдельного изменения аппаратной спецификации проекта.

3. Зафиксированное программное окружение

Для данного baseline используются:

STM32CubeIDE: 2.1.1
STM32CubeF4:  1.28.3
Target MCU:   STM32F407VETx
Toolchain:    GNU Tools for STM32
Language:     C / C++

Конфигурирование MCU выполняется через STM32CubeMX.

Общая схема работы:

STM32CubeMX
     ↓
brew_controller.ioc
     ↓
Generate Code
     ↓
STM32CubeIDE
     ↓
Build / Debug

Разработчикам запрещено без отдельного тикета обновлять STM32CubeF4 или мигрировать .ioc на другую firmware-версию.

4. Расположение проекта

Рекомендуемый корень репозитория:

brew-controller/

STM32-проект располагается в:

brew-controller/firmware/

Рабочий каталог STM32CubeIDE рекомендуется хранить отдельно от Git-репозитория.

Пример:

D:\Projects\brew-controller\

и:

D:\STM32Workspace\BrewController\

Не рекомендуется помещать Eclipse workspace .metadata внутрь репозитория.

5. Структура репозитория

После завершения GT-00 ожидается структура:

brew-controller/
│
├── firmware/
│   ├── Core/
│   │   ├── Inc/
│   │   └── Src/
│   │
│   ├── Drivers/
│   │   ├── CMSIS/
│   │   └── STM32F4xx_HAL_Driver/
│   │
│   ├── BSP/
│   ├── Platform/
│   ├── Services/
│   ├── Domain/
│   ├── App/
│   ├── GUI/
│   │
│   ├── brew_controller.ioc
│   ├── startup_stm32f407xx.s
│   ├── STM32F407VETX_FLASH.ld
│   ├── .project
│   └── .cproject
│
├── docs/
├── tests/
├── tools/
├── scripts/
├── .gitignore
└── README.md

На A-001 каталоги Clean Architecture могут быть еще пустыми или создаваться следующими задачами GT-00.

6. Настройки CubeMX для A-001

MCU

Выбрать:

STM32F407VETx

Проверить:

Series:       STM32F4
Line:         STM32F407/417
Package:      LQFP100
Target MCU:   STM32F407VETx

SYS

Открыть:

System Core
→ SYS

Установить:

Debug = Serial Wire

Для SWD резервируются:

PA13 → SWDIO
PA14 → SWCLK

HAL Timebase

На A-001 оставить:

SysTick

FreeRTOS на этом этапе не подключается.

RCC

На A-001 окончательная настройка PLL и 168 MHz не выполняется.

Настройка:

HSE
PLL_M
PLL_N
PLL_P
PLL_Q
AHB
APB1
APB2
Flash Latency

будет выполнена в A-002 — Configure clocks.

7. Периферия, запрещенная на A-001

На данном этапе специально не настраиваются:

ADC;

DMA;

SPI;

I2C;

FSMC/FMC;

дисплей;

STemWin;

FreeRTOS;

PID;

NTC;

SSR;

насос;

Recipe Engine;

Beer FSM;

Process FSM.

A-001 является минимальным baseline-проектом.

8. Настройки Project Manager

Project Name

Использовать:

brew_controller

Не использовать пробелы, кириллицу и специальные символы.

Toolchain / IDE

Использовать:

STM32CubeIDE

Firmware Package

Использовать зафиксированную версию:

STM32CubeF4 1.28.3

Не выполнять автоматическую миграцию проекта на более новую версию без отдельного решения команды.

9. Настройки Code Generator

Рекомендуется включить:

Keep User Code when re-generating

Также для большого проекта рекомендуется:

Generate peripheral initialization as a pair
of '.c/.h' files per peripheral

Это позволит в последующих этапах получать отдельные:

gpio.c / gpio.h
adc.c  / adc.h
tim.c  / tim.h
usart.c / usart.h

и не раздувать main.c.

10. Generated Code Policy

Код CubeMX и ST считается generated/vendor code.

Основная бизнес-логика проекта не должна размещаться в:

main.c
stm32f4xx_it.c
stm32f4xx_hal_msp.c
Drivers/STM32F4xx_HAL_Driver/

Если изменение generated-файла необходимо, собственный код допускается только внутри блоков:

/* USER CODE BEGIN ... */

/* USER CODE END ... */

Главные модули проекта должны находиться в собственных каталогах:

BSP/
Platform/
Services/
Domain/
App/
GUI/

11. Запрещено изменять STM32 HAL

Не редактировать вручную:

Drivers/STM32F4xx_HAL_Driver/

Например, запрещено добавлять прикладные функции в:

stm32f4xx_hal_gpio.c
stm32f4xx_hal_adc.c
stm32f4xx_hal_tim.c

Своя функциональность должна реализовываться выше HAL.

Будущая схема зависимостей:

Application
    ↓
Domain / Services
    ↓
Drivers
    ↓
BSP
    ↓
STM32 HAL

12. Минимальный startup flow

На этапе A-001 ожидается примерно такой поток запуска:

RESET
  ↓
startup_stm32f407xx.s
  ↓
SystemInit()
  ↓
main()
  ↓
HAL_Init()
  ↓
SystemClock_Config()
  ↓
Peripheral Init
  ↓
while (1)

Функциональность технологического процесса пока отсутствует.

13. Проверка startup-файла

В проекте должен присутствовать:

startup_stm32f407xx.s

Если проект содержит startup другого семейства, например:

startup_stm32f401xx.s
startup_stm32f411xx.s

необходимо проверить выбранный MCU.

14. Проверка linker script

В проекте должен использоваться linker script для STM32F407VETx, например:

STM32F407VETX_FLASH.ld

Необходимо проверить секцию:

MEMORY

и убедиться, что она соответствует выбранному варианту MCU.

15. Проверка compiler target

Для STM32F407 ожидается Cortex-M4.

В настройках compiler должны присутствовать параметры, соответствующие целевому MCU, например:

-mcpu=cortex-m4

Для аппаратного FPU типовая конфигурация содержит:

-mfpu=fpv4-sp-d16
-mfloat-abi=hard

Также ожидаются определения:

STM32F407xx
USE_HAL_DRIVER

Не следует вручную менять эти параметры, если проект был корректно сгенерирован CubeMX.

16. Первая сборка

В STM32CubeIDE выполнить:

Project
→ Clean...

Затем:

Project
→ Build Project

или:

Ctrl+B

Ожидаемый результат:

Build Finished
0 compilation errors

Основным артефактом A-001 является:

brew_controller.elf

На данном этапе отсутствие .bin или .hex не считается ошибкой.

Их автоматическая генерация будет настроена отдельной задачей GT-00.

17. Git policy

Generated build-каталоги не должны попадать в репозиторий.

Минимальный .gitignore:

**/Debug/
**/Release/
**/build/

**/*.o
**/*.d
**/*.su
**/*.cyclo
**/*.log

.metadata/

На текущем этапе не следует исключать:

.project
.cproject
.settings/
*.ioc
*.ld
startup*.s

если команда не приняла отдельную стратегию воспроизводимой CMake-сборки.

18. Что обязательно хранить в Git

Минимально:

firmware/Core/
firmware/Drivers/
firmware/brew_controller.ioc
firmware/.project
firmware/.cproject
firmware/*.ld
firmware/startup*.s

Файл .ioc считается частью исходного проекта.

19. Рекомендуемая Git-ветка

Для A-001:

feature/gt00-a001-baseline-project

Базовая ветка:

develop

Схема:

develop
   ↓
feature/gt00-a001-baseline-project
   ↓
implementation
   ↓
local build
   ↓
tests
   ↓
Pull Request
   ↓
review
   ↓
QA
   ↓
develop

Прямые push в main запрещены.

20. Commit

Рекомендуемый commit:

feat(platform): create STM32F407VE baseline project

21. Pull Request

PR должен ссылаться на:

GT-00 / A-001

Пример содержания:

Ticket:
GT-00 / A-001

Changes:
- Added STM32F407VETx CubeMX project
- Added CubeIDE project metadata
- Added startup and linker files
- Added STM32CubeF4 1.28.3 HAL/CMSIS
- Enabled Serial Wire debug
- Added initial repository ignore rules

Tests:
- Clean build: PASS
- MCU verification: PASS
- IOC open/regenerate: PASS

Risks:
- Clock configuration is intentionally deferred to A-002

22. Developer A — checklist

Перед созданием PR проверить:

[ ] STM32CubeIDE 2.1.1 используется.

[ ] STM32CubeMX запускается.

[ ] STM32F407VETx выбран.

[ ] Package = LQFP100.

[ ] SYS → Debug → Serial Wire.

[ ] PA13 зарезервирован под SWDIO.

[ ] PA14 зарезервирован под SWCLK.

[ ] SysTick используется как HAL timebase.

[ ] STM32CubeF4 1.28.3 зафиксирован.

[ ] Project name = brew_controller.

[ ] brew_controller.ioc сохранен.

[ ] ADC не настроен.

[ ] DMA не настроен.

[ ] FreeRTOS не подключен.

[ ] STemWin не подключен.

[ ] TFT не подключен.

[ ] SSR не подключен.

[ ] Pump не подключен.

[ ] Clean проходит.

[ ] Build проходит.

[ ] 0 compilation errors.

[ ] ELF создан.

[ ] startup соответствует STM32F407.

[ ] linker script соответствует STM32F407VETx.

[ ] Debug/ отсутствует в Git.

[ ] .ioc присутствует в Git.

23. QA tests

TC-A001-001 — Baseline Project Builds

Цель: проверить чистую сборку проекта.

Действия:

1. Получить чистую копию репозитория.
2. Открыть firmware-проект.
3. Выполнить Clean.
4. Выполнить Build.

Ожидается:

PASS

0 compilation errors
ELF generated

TC-A001-002 — MCU Verification

Проверить:

Target MCU = STM32F407VETx
Package    = LQFP100

Проверить наличие:

startup_stm32f407xx.s

и корректного linker script.

Ожидается:

PASS

TC-A001-003 — Minimal Peripheral Configuration

Проверить, что на A-001 не подключены:

ADC
DMA
SPI
FSMC/FMC
FreeRTOS
STemWin

Ожидается:

PASS

TC-A001-004 — IOC Validation

Проверить файл:

brew_controller.ioc

Действия:

1. Открыть .ioc в STM32CubeMX.
2. Убедиться в отсутствии ошибок проекта.
3. Сохранить проект.

Ожидается:

PASS

TC-A001-005 — Code Regeneration

Действия:

1. Зафиксировать git status.
2. Открыть brew_controller.ioc.
3. Выполнить Generate Code.
4. Собрать проект.
5. Проверить git diff.

Ожидается:

Build: PASS

Не должно возникать необъяснимых массовых изменений.

TC-A001-006 — Second Environment Build

Developer B или QA выполняет сборку на отдельной рабочей среде.

Действия:

git clone
open project
clean
build

Ожидается:

PASS

Цель теста — подтвердить воспроизводимость baseline-проекта.

24. Definition of Done

A-001 закрывается только при выполнении всех условий:

[ ] Рабочий каталог проекта создан.

[ ] STM32CubeIDE workspace находится отдельно от Git.

[ ] STM32CubeIDE 2.1.1 проверен.

[ ] STM32CubeMX установлен.

[ ] STM32F407VETx выбран.

[ ] LQFP100 выбран.

[ ] SYS Debug = Serial Wire.

[ ] STM32CubeF4 1.28.3 используется.

[ ] Проект называется brew_controller.

[ ] .ioc сохранен.

[ ] CubeMX успешно генерирует код.

[ ] CubeIDE успешно открывает проект.

[ ] Clean выполняется.

[ ] Build выполняется.

[ ] Ошибок компиляции нет.

[ ] ELF создается.

[ ] startup соответствует STM32F407.

[ ] linker script соответствует STM32F407VETx.

[ ] ADC не настроен.

[ ] DMA не настроен.

[ ] FreeRTOS не подключен.

[ ] STemWin не подключен.

[ ] TFT не подключен.

[ ] SSR не подключен.

[ ] Pump не подключен.

[ ] .gitignore настроен.

[ ] Build artifacts не попадают в Git.

[ ] brew_controller.ioc попадает в Git.

[ ] Проект собирается после чистого checkout.

[ ] TC-A001-001 пройден.

[ ] TC-A001-002 пройден.

[ ] TC-A001-003 пройден.

[ ] TC-A001-004 пройден.

[ ] TC-A001-005 пройден.

[ ] TC-A001-006 пройден.

[ ] Pull Request прошел review.

[ ] QA подтвердил готовность этапа.

25. Что намеренно НЕ входит в A-001

Следующие задачи выполняются позже:

A-002

Настройка тактирования STM32F407VE
168 MHz SYSCLK
HSE
PLL
AHB
APB1
APB2
Flash Latency
MCO verification

Следующие GT

ADC + DMA
5 × NTC 10K
STemWin
TFT
SSR Heater
Pump
PID
Safety Manager
Beer Process
Distillation Process
Recipes
Storage
CI/HIL

26. Следующий этап

После успешного закрытия A-001 перейти к:

A-002 — Configure STM32F407VE clocks

Целевая конфигурация:

HSE
  ↓
PLL
  ↓
SYSCLK = 168 MHz

AHB  = 168 MHz
APB1 = 42 MHz
APB2 = 84 MHz

На A-002 необходимо отдельно проверить:

частоту установленного на конкретной плате HSE;

PLL_M;

PLL_N;

PLL_P;

PLL_Q;

AHB prescaler;

APB1 prescaler;

APB2 prescaler;

Flash latency;

реальные частоты таймеров;

MCO для аппаратной проверки частоты.

27. Статус этапа

До прохождения QA:

GT-00 / A-001
STATUS: IN PROGRESS

После успешного review и TC-A001-001...006:

GT-00 / A-001
STATUS: DONE

28. Идентификация baseline

Рекомендуемая версия инфраструктуры после завершения всего GT-00:

v0.0.1-infrastructure

A-001 является первым firmware baseline, на котором будут строиться все следующие аппаратные и программные модули проекта.
