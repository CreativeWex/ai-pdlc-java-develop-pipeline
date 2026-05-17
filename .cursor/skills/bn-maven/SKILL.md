---
name: bn-maven
description: Сборка, тесты и анализ зависимостей Maven-проектов (mvn/mvnw, pom.xml, multi-module). ВЫЗЫВАЙ ВСЕГДА при при намерении собрать проект, запустить тесты, построить дерево зависимостей, диагностировать ошибки сборки, а также при словах Maven, mvn, pom, сборка, compile, package, dependency tree, /bn-maven.
---

# Maven: сборка, тесты, зависимости
Скилл обязателен при любой работе с Maven в репозитории. Следуй ему полностью, не полагаясь на память.

# Ограничения - тебе НИКОГДА нельзя нарушать эти ограничения
1. **Запрещено без явной просьбы пользователя:** `mvn deploy`, `mvn install` в локальный репозиторий «на всякий случай», `dependency:purge-local-repository`, массовое `versions:set`, `clean` с удалением вне `target/`.
2. **Запрещено без явной просьбы пользователя:** менять `settings.xml`

## Выбор Maven
**ВСЕГДА** проверяй наличие `mvnw` в корне проекта:
- Если есть `./mvnw` (Unix) или `mvnw.cmd` (Windows) — используй ТОЛЬКО его
- Если нет — используй системный `mvn`
- Никогда не игнорируй wrapper, даже если он старый

## SNAPSHOT зависимости
**Правила:**
1. Перед сборкой с SNAPSHOT — **всегда** проверяй `mvn dependency:get -Dartifact=groupId:artifactId:version-SNAPSHOT`
2. Если SNAPSHOT не находится — НЕ предлагай убрать `-SNAPSHOT`, а запроси у пользователя правильный репозиторий
3. При подозрении на старый SNAPSHOT — `mvn clean install -U` (только после подтверждения)
4. Никогда не исправляй ошибку "Can't find SNAPSHOT" добавлением `-SNAPSHOT` вручную

## Лучшие практики pom.xml
1. **Используй Parent / BOM** — версии Spring Boot и общих библиотек в `dependencyManagement`, НЕ ДУБЛИРУЙ версии в каждом `<dependency>`.
2. ВСЕ ВЕРСИИ ЗАВИСИМОСЕЙ (dependency versions) и версии плагинов (plugin versions) ***ДОЛЖНЫ быть объявлены в блоке <properties>*** в виде переменных.
Правильная конструкция:
```xml
<!-- ✅ ПРАВИЛЬНО -->
<properties>
    <java.version>17</java.version>
    <spring-boot.version>3.1.5</spring-boot.version>
    <lombok.version>1.18.30</lombok.version>
    <testcontainers.version>1.19.1</testcontainers.version>
    <maven-compiler-plugin.version>3.11.0</maven-compiler-plugin.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <version>${spring-boot.version}</version>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>${lombok.version}</version>
        <optional>true</optional>
    </dependency>
</dependencies>
```
Неправильная конструкция
```xml
<!-- ❌ ЗАПРЕЩЕНО - хардкод версий -->
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <version>3.1.5</version>  <!-- ❌ жесткая версия -->
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>1.18.30</version>  <!-- ❌ жесткая версия -->
    </dependency>
</dependencies>
```
3. НЕ ТЯНИ тестовые зависимости в compile, ЯВНО УКАЗЫВАЙ `<scope>test</scope>` для ВСЕХ тестовых зависимостей
4. **Плагины** — фиксируй версии `maven-compiler-plugin`, `maven-surefire-plugin`, `spring-boot-maven-plugin`.

## Дерево зависимостей
1. `mvn dependency:tree` — обзор.
2. При конфликте версий — `mvn dependency:tree -Dverbose` и ищи `omitted for conflict`.
3. Сузь scope: `-Dscope=compile` или `test`.
4. Для Spring Boot — проверь `spring-boot-dependencies` / BOM в `dependencyManagement`.
5. Не «лечи» конфликт случайным `<version>` в `<dependency>` без анализа — предпочитай BOM, `dependencyManagement` родителя или `<exclusions>` с обоснованием.

## Многомодульные проекты

### Сборка конкретного модуля
```bash
# Собрать модуль и его зависимости (но не все модули)
mvn clean install -pl :module-artifact-id -am

# Собрать модуль без родительских сборок
mvn clean compile -pl . -am -amd

# Пропустить сборку модуля
mvn clean install -pl '!module-to-skip'
```

## Очистка и обновление зависимостей
### Симптомы битого локального кэша:
- `Could not resolve` для артефакта, который точно есть в Nexus. ВСЕГДА проси пользователя удостовериться, что искомая версия присутствует в Nexus. НИКОГДА не решай сам, ВСЕГДА спрашивай пользователя
- Неожиданные версии классов в runtime
- Ошибки сборки после обновления зависимостей

### Решения:

**Уровень 1: Обновить snapshots**
```bash
mvn clean install -U  # -U = update snapshots
```
**Уровень 2: Очистить конкретную зависимость**
```bash
# Удалить из локального репозитория
rm -rf ~/.m2/repository/com/example/problematic-lib

# Или через maven
mvn dependency:purge-local-repository -DmanualInclude=groupId:artifactId
```
**Уровень 3: Полная перезагрузка (крайний случай)**
Перед выполнением этого шага ВСЕГДА получай разрешение пользователя. Тебе ЗАПРЕЩАЕТСЯ выполнять этот шаг без явного разрешения пользователя
```bash
rm -rf ~/.m2/repository
mvn clean install
```

## Ошибки Classpath в runtime

### Проблема: NoClassDefFoundError (класс есть) или ClassNotFoundException (класс отсутствует)

**Диагностика:**
```bash
# 1. Проверь, в какой JAR должен быть класс
mvn dependency:tree | grep -i "library-name"

# 2. Посмотри реальный classpath при запуске
mvn exec:java -Dexec.mainClass="com.example.Main" -X | grep -i classpath

# 3. Проверь, что JAR действительно собран
jar tf target/app.jar | grep "MissingClass"

# 4. Для Spring Boot - проверь fat JAR
mvn spring-boot:run -Dspring-boot.run.arguments="--debug"
```
**Решение**:
- *Зависимость в неправильном scope* - измени `test`/`provided` на `compile`
- *Transitive dependency excluded* - верни явно или убери exclusion
- *Fat JAR конфликт* - настрой maven-shade-plugin с переименованием

## Формат ответа после Maven
Кратко сообщи пользователю:
```markdown
**Команда:** `mvn ...` (каталог: `...`)
**Результат:** BUILD SUCCESS | FAILURE
**Тесты:** run / failures / errors (если применимо)
**Действие:** что исправлено или что нужно от пользователя
```
При **FAILURE** — приведи ключевую строку ошибки (не весь лог на 500 строк).
