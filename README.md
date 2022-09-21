Модель связи между элементами представлена ниже:
```mermaid
graph TD;
    user[Пользователь]
    acc[Аккаунт]
    session[Сессия]
    token[Токен]

    vendor[Продавец]
    buyer[Покупатель]

    bet[Ставка]
    lot[Лот]
    tag[Тэг]

    acc -- Содержит 1:1 --> session 
    acc -- Принадлежит M:1 --> user
    session -- Содержит 1:1 --> token

    user -- Является 1:1 --> vendor & buyer

    lot -- Принадлежит M:1 --> vendor
    buyer -- Имеет 1:М --> bet
    bet -- Принадлежит M:1 --> lot

    lot -- Содержит N:M --> tag 
    tag -- Содержит M:N --> lot 
```

Для описания полей и полной системы организации был применен язык моделирования баз данных `Prisma`. Все модели представлены ниже. 

> Ниже не подразумевается использование какой либо конкретной СУБД, все модели в целом применимы в документной, графовой или в реляционной базе. 

## Пользователь и система аутентификации

Модель пользователя имеет следующий вид:
```ts
model User {
    id            String    @id @default(uuid())
    name          String?
    email         String?   @unique
    emailVerified String
    imageUrl      String?
    accounts      Account[]
    sessions      Session[]
    vendor        Vendor?
    buyer         Buyer?
}
```

Сам факт того, что он является продавцом или покупателем отражен в связи с одним элементом соответствующих объектов `Vendor` и `Buyer`. Эта связь имеет тип 1:1. При этом у одного пользователя может быть несколько сессий и несколько аккаунтов.

Для входа в систему имеется токены типа OAuth и обычный токен для сессии. Так что таблицы сессии и аккаунта имеют вид:
```ts
model Session {
    id           String   @id @default(uuid())
    sessionToken String   @unique
    expires      DateTime
    user         User 
    userId       String
}
```

```ts
model Account {
    id                 String  @id @default(uuid())
    accType            String
    provider           String
    providerAccountId  String
    refresh_token      String?
    access_token       String?
    expires_at         Int?
    token_type         String?
    scope              String?
    id_token           String?
    session_state      String?
    oauth_token_secret String?
    oauth_token        String?
    user User 
}
```

У пользователя может быть несколько аккаунтов, чтобы поддерживать возможность входа через несколько систем авторизации, например через токен OAuth Google и т.д.

Сессия определяет состояние входа на одном устройстве. Так соединение получается защищенным и предотвращает взлом при краже токена и запуске его на другом устройстве.

## Модели покупателей и продавцов
Как было отмечено, пользователь ссылается на элементы множества покупателей и продавцов (причем на одного в каждом): 
```ts
model User{
    ...
    vendor        Vendor?
    buyer         Buyer?
}
```
Факт связи с элементом продавца указывает на то, что он имеет соответствующую роль. Аналогично с элементом множества покупателей. Таким образом пользователь может иметь несколько ролей. Например, если в интерфейсе пользователь указал, что хочет начать продажу, то для него создается объект Vendor, на который ссылается его объект User и в нем далее хранится информация о лотах, что он запустил.

Соответственно объект продавца имеет вид:
```ts
model Vendor {
    id     String @id @default(uuid())
    user   User
    lot    Lot[]
}
```
В ней есть указание на созданные лоты и на пользователя, которым данный продавец является. 

Модель покупателя имеет похожий вид, за исключением типа связи M:N с лотами через дополнительную модель (например, это может быть промежуточная таблица в SQL) - `Bet` - ставка. 

```ts
model Buyer {
    id     String @id @default(uuid())
    user   User  
    bet    Bet[]
}
```

Ставка хранит информацию и связывает лот с пользователем:
```ts
model Bet {
    id        String   @id @default(uuid())
    createdAt DateTime @default(now())
    updatedAt DateTime @updatedAt
    current   Boolean  @default(true)
    canceled  Boolean  @default(false)
    price     Float

    buyer     Buyer
    lot       Lot 
}
```
Здесь: 
1. `сurrent` - отражает то, что ставка является текущей и не перезаписана новой ставкой того же пользователя на том же лоте;
2. `canceled` - показывает, отменена ли ставка;
3. `price` - цена в ставке

Так же в модели содержится информация о времени создания и изменения.
## Лот и теги
Модель лота:
```ts
model Lot {
    id           String   @unique @default(uuid())
    description  String
    bet          Bet[]
    tags         Tag[]
    createdAt    DateTime @default(now())
    updatedAt    DateTime @updatedAt
    endDate      DateTime
    closed       Boolean  @default(false)
    minPrice     Float
    currentPrice Float
    vendor       Vendor   
}
```
У него есть ссылка на ставки, информация о том, закрыт ли он, и связь с одним продавцом. Так же поддерживается связь M:N с таблицей тегов: 
```ts
model Tag {
    id   String @unique @default(uuid())
    lots Lot[]
    name String
}
```
Факт того, что оба объекта содержат `Lot[]` и `Tag[]` отражает связь кардинальность связи M:N.
