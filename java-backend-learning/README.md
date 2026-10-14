# Java Backend Learning Sprint

Навчальний спринт на базі [CommaFeed](commafeed) 7.3.2.

## Крок 1. Середовище

Версії, зняті на початку кроку:

| Інструмент | Версія |
|---|---|
| Java | 21.0.10 LTS (`C:\Program Files\Java\jdk-21.0.10`) |
| Maven | 3.9.16, через `mvnw` / `mvnw.cmd` |
| Docker | 29.8.2 |
| Git | 2.55.0.windows.3 |

CommaFeed 7.3.2 з цього коміту збирається JDK 25 (`maven.compiler.release` = 25). Зараз `java` у `Path` і Java, яку бачить `mvnw.cmd -version`, це **25.0.4.1** LTS (`C:\Program Files\Java\jdk-25.0.4.1`). JDK 21 лишається встановленим, але для `commafeed-server` його замало: `javac` відповідає `release version 25 not supported`.

ОС: Windows 10. Команди нижче розраховані на CMD; у PowerShell ті самі `mvnw.cmd`.
