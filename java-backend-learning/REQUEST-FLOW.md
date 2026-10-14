# `GET /rest/user/profile`

Шлях складається з `@Path("/rest/user")` на класі і `@Path("/profile")` плюс `@GET` на методі. Порт сервера заданий у `commafeed-server/src/main/resources/application.properties`, рядок 2: `quarkus.http.port=8082`.

Нижче шлях уже залогіненого браузера: Quarkus form-cookie, потім метод, потім база. Код застосунку не викликає `new UserREST()`: екземпляр один на застосунок, його збирає Arc за `@Singleton`.

| Клас | Метод | Що робить | Що повертає |
|---|---|---|---|
| HTTP-шар Quarkus (Vert.x) | прийом запиту | Слухає порт 8082. Класу в репозиторії немає, це рантайм `quarkus-rest`. Бачить `GET /rest/user/profile` і віддає його в ланцюжок автентифікації | `RoutingContext` |
| Form-механізм Quarkus | читання cookie | Увімкнений у `application.properties`, рядки 19–22 (`quarkus.http.auth.form.enabled=true`). Розшифровує http-only cookie сесії. У cookie лежить id користувача, записаний під час логіну | `TrustedAuthenticationRequest` з цим id. Якщо cookie немає або вона бита, запит лишається анонімним |
| `TrustedIdentityProvider` | `authenticate` | Файл `security/identity/TrustedIdentityProvider.java`, рядки 41–61. Перетворює principal на `Long` і в транзакції `UnitOfWork` вантажить користувача, ролі і збирає особу | `SecurityIdentity`: principal дорівнює id рядком, ролі `USER` і за потреби `ADMIN` |
| `UserDAO` → `GenericDAO` | `findById` | `UserDAO.java` рядки 14–16 передає `User.class`. `GenericDAO.findById`, рядки 45–47, викликає `entityManager.find`. Hibernate читає таблицю `USERS` (`User.java`, рядок 17) за `id` | `User`. Якщо рядка немає, `AuthenticationFailedException` |
| `PostLoginActivities` | `executeFor` | Рядки 21–30. Якщо `lastLogin` порожній або старший за 30 хвилин, оновлює його через `userDAO.merge` | нічого. Інакше виходить одразу |
| `UserService` | `getRoles` | Рядок 165–167. Делегує в DAO | `Set<Role>` |
| `UserRoleDAO` | `findRoles` → `findAll` | Рядки 28–33. QueryDSL: `USERROLES` де `user_id` дорівнює id користувача (`UserRole.java`, рядки 16 і 28) | Імена ролей, які `TrustedIdentityProvider` кладе в `SecurityIdentity` |
| Перевірка `@RolesAllowed` | до методу | Анотація на `UserREST`, рядок 75: `@RolesAllowed(Roles.USER)`. Quarkus звіряє ролі особи з константою `Roles.USER` (`Roles.java`, рядок 7) | Дозвіл увійти в метод, або відповідь 403 без виклику `getUserProfile` |
| Arc | створення біна | `@Singleton` на `UserREST`, рядок 80. Конструктор дописує Lombok `@RequiredArgsConstructor` (рядок 79) з `final`-полів, включно з `UserDAO` і `UserService`. `new` у вихідному файлі немає | Той самий екземпляр `UserREST` на весь застосунок |
| `UserREST` | `getUserProfile` | Рядки 269–288. Збирає `UserModel`: id, name, email, enabled, apiKey, lastForceRefresh. Пароль у модель не копіюється. `@Transactional` тримає обидва читання нижче в одній транзакції. `@Produces(APPLICATION_JSON)` на рядку 76 каже віддати JSON | `UserModel` |
| `AuthenticationContext` | `getCurrentUser` | Рядки 19–29. Бере `securityIdentity.getPrincipal().getName()` (знову id) і йде в DAO. Аноніма сюди вже не пускає `@RolesAllowed` | `User` |
| `UserDAO` → `GenericDAO` | `findById` | Друге читання `USERS` за тим самим id, уже в транзакції ендпоінта. Перше було в `TrustedIdentityProvider` | `User` для полів профілю |
| `UserRoleDAO` | `findAll(User)` | Рядки 28–29. Той самий фільтр `USERROLES.user_id`, тепер щоб виставити `userModel.admin`, якщо роль `ADMIN` | `List<UserRole>` |
| Jackson (`quarkus-rest-jackson`) | серіалізація | Перетворює повернутий `UserModel` (`frontend/model/UserModel.java`) на тіло відповіді | HTTP 200 і JSON профілю |

`POST /rest/user/profile` сюди не входить: той самий шлях, інший метод, `saveUserProfile`.

Якщо замість cookie в query є `apiKey` і метод саме `GET`, до `getUserProfile` веде інша гілка. `ApiKeyAuthenticationMechanism.authenticate` (`security/mechanism/ApiKeyAuthenticationMechanism.java`, рядки 22–36) віддає ключ у `DatabaseApiKeyIdentityProvider`. Той логінить через `UserService.login(apiKey)` і теж будує `SecurityIdentity` з id. Далі таблиця з рядка `@RolesAllowed` та сама.
