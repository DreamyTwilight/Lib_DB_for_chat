# lib_db_for_chat
Library for creating and managing a SQLite database for chat applications.

Source code of the libdb in the project: https://github.com/ya-masterskaya-cpp/q2_2025_chat_project_team_2

## sqlite_bd

Минималистичная библиотека-обёртка для работы с `SQLite3` на `C++17`. Кроссплатформенность: Windows и Linux
`Conan` + `CMake`, автоматические тесты `Catch2`
Предназначена для хранения данных IRC-чата: истории сообщений, пользователей, комнат и т.д. 
Отдельно офомрлены функции работы со временем `time_utils.hpp`.

### Быстрая сборка (MS Visual Studio)

```bash
rm -rf build
conan install . --output-folder=build --build=missing -s build_type=Debug
# сборка без тестовой части с ключем -DBUILD_TESTING=OFF
cmake -B build -G "Visual Studio 17 2022" -A x64 -DCMAKE_TOOLCHAIN_FILE=build/conan_toolchain.cmake
cmake --build build --config Debug
```
### Структура проекта
<pre>
libdb/
├── include/        
|    ├── db.hpp
|    └── time_utils.hpp
├── src/            
|    ├── db.cpp
|    └── stmt.hpp
├── CMakeLists.txt  
├── conanfile.txt 
└── test/           
     └── test.cpp </pre>

При отсутствии файла БД, будет создан файл БД с исходной структурой БД. Имя файла по умолчанию: chat.db </br>
Имеется возможность задать имя файла БД. </br>

Униальные поля: логин пользователя, наименование комнаты и роль пользователя в БД. </br>

</br>
## Предполагаемый функционал сервера в части работы с БД:
При загрузке:
- запрашивает все комнаты со списком зарегистрированных пользовтелей для каждой комнаты (`GetAllRoomWithRegisteredUsers`)
- запрашивает всех пользователей (`GetAllUsers`)
- запрашивает по всем комнатам номера последних сообщений (`GetAllUsers`)
  
Во время работы:
- контролирует уникальность логинов и имен пользователей, наименований комнат
- генерирует номера сообщений для каждой комнаты (сообщения комнаты имеют свою, уникальную в рамках комнаты, непрерывную нумерацию, хранимую в поле `id_message_in_room` таблицы `messages`) для организации контроля одинакового списка сообщений у пользователей одной комнаты, предполается, что при наличии пропусков в нумерации приложение клиента запрашивает недостающее
- фиксирует время создания пользователей и комнат, получения сообщений </br>

## Используемые структуры (`namespace db`)
```cpp
struct User {
    std::string login;
    std::string name;
    std::string password_hash;
    std::string role;
    bool is_deleted;
    int64_t unixtime; //наносекунды
};
struct Message {
    std::string message;
    int64_t unixtime; //наносекунды
    std::string user_login;
    std::string room;
    int64_t id_message_in_room;
};
```
</br>

## Перечень функций работы с БД (`namespace db`)

#### 1. Управление подключением
``` cpp
    bool OpenDB(); // открывает соединение с БД, инициализирует схему (если БД новая)

    void CloseDB(); // закрывает соединение

    std::string GetVersionDB(); //возвращает внутренний номер версии БД
```
#### 2. Управление пользователями
``` cpp
    bool CreateUser(const User& user); // добавляет пользователя

    bool DeleteUser(const std::string& user_login); // удаляет пользователя, если он не состоит
                                                      // в комнатах (иначе только is_deleted = true)

    bool IsUser(const std::string& user_login); // проверяет существование пользователя

    bool ChangeUserName(const std::string& user_login, const std::string& new_name); // меняет имя пользователя

    bool IsAliveUser(const std::string& user_login); // проверяет, что пользователь существует и не помечен на удаление

    std::optional<User> GetUserData(const std::string& user_login); // возвращает информацию по отдельному пользователю

    std::vector<User> GetAllUsers(); // возвращает всех пользователей (включая удаленных)

    std::vector<User> GetActiveUsers(); // возвращает только активных (is_deleted = false)

    std::vector<User> GetDeletedUsers(); // возвращает удаленных (is_deleted = true)
```
#### 3. Управление комнатами
``` cpp
    bool CreateRoom(const std::string& room, int64_t unixtime); // создает комнату

    bool DeleteRoom(const std::string& room); // удаляет комнату, сообщения комнаты из БД и связи пользователей с конатой

    bool ChangeRoomName(const std::string& current_room_name, const std::string& new_room_name); // переименование комнаты

    bool IsRoom(const std::string& room); // проверяет существование комнаты

    std::vector<std::string> GetRooms(); // возвращает список всех комнат
```
#### 4. Связи пользователей и комнат
``` cpp
    bool AddUserToRoom(const std::string& user_login, const std::string& room); // добавляет пользователя в комнату

    bool DeleteUserFromRoom(const std::string& user_login, const std::string& room); // удаляет пользователя из комнаты

    std::vector<std::string> GetUserRooms(const std::string& user_login); // возвращает комнаты пользователя

    std::vector<User> GetRoomActiveUsers(const std::string& room); // возвращает активных пользователей комнаты

    // возвращает все комнаты со списком зарегистрированных пользовтелей для каждой комнаты
    std::unordered_map<std::string, std::unordered_set<std::string>> GetAllRoomWithRegisteredUsers();
```
#### 5. Управление сообщениями
``` cpp
    bool InsertMessageToDB(const Message& message); // добавляет сообщение в комнату

    // получение сообщений комнаты по id, если указать одинаковый id вместо диапазоно, то получим одно сообщение
    // не уверен в необходимости отдельного метода для получения одного сообщения
    std::vector<Message> GetRangeMessagesRoom(const std::string& room, int64_t id_message_begin, int64_t id_message_end);

    int GetCountRoomMessages(const std::string& room); // возвращает количество сообщений в комнате
```

### Функции работы со временем (`namespace utime`)
```cpp
    inline int64_t GetUnixTimeNs();  // получение unix времени с точностью до наносекунды

    // потокозащищенная функция преобразования наносекунд в дату ("2023-11-15") и время ("14:30:45")
    inline std::pair<std::string, std::string> UnixTimeToDateTime(int64_t unix_time_ns);
```
</br>

## Структура таблиц БД

#### `ТАБЛИЦА metadata` </br>
- `key` (TEXT, PRIMARY KEY) – ключ (например, schema_version).</br>
- `value` (TEXT) – значение.</br>

#### `ТАБЛИЦА roles`
- `roles_id` (INTEGER, PRIMARY KEY) – ID роли.
- `role` (TEXT, UNIQUE) – название роли (admin, user).

#### `ТАБЛИЦА rooms`
- `rooms_id` (INTEGER, PRIMARY KEY) – ID комнаты.
- `room` (TEXT, UNIQUE) – название комнаты.
- `unixtime` (INTEGER) – время создания (наносекунды).

#### `ТАБЛИЦА users`
- `users_id` (INTEGER, PRIMARY KEY) – ID пользователя.
- `login` (TEXT, UNIQUE) – логин.
- `name` (TEXT) – имя.
- `password_hash` (TEXT) – хеш пароля.
- `roles_id` (INTEGER, FOREIGN KEY) – ссылка на roles.roles_id.
- `is_deleted` (BOOLEAN) – флаг удаления.
- `unixtime` (INTEGER) – время регистрации (наносекунды).

#### `ТАБЛИЦА messages`
- `messages_id` (INTEGER, PRIMARY KEY) – ID сообщения.
- `message` (TEXT) – текст.
- `unixtime` (INTEGER) – время отправки (наносекунды).
- `users_id` (INTEGER, FOREIGN KEY) – отправитель (users.users_id).
- `rooms_id` (INTEGER, FOREIGN KEY) – комната (rooms.rooms_id, ON DELETE CASCADE).
- `date` (TEXT) – дата в формате YYYY-MM-DD.
- `time` (TEXT) – время в формате HH:MM:SS.
- `id_message_in_room` (INTEGER) – номер сообщения в комнате.

#### `ТАБЛИЦА user_rooms`
- `user_rooms_id` (INTEGER, PRIMARY KEY) – ID связи.
- `users_id` (INTEGER, FOREIGN KEY, ON DELETE CASCADE) – пользователь.
- `rooms_id` (INTEGER, FOREIGN KEY, ON DELETE CASCADE) – комната.

UNIQUE(users_id, rooms_id) – запрет дублирования связей.
</br>

## Ключевые зависимости

Каскадное удаление: </br>
Удаление комнаты → автоматически удаляет записи в user_rooms и messages.
</br>

## Планы
Добавить:
- метод получения номера последнего сообщения комнаты.
- метод получения списка комнат и номеров последних сообщений комнат для подгрузки при запуске сервера
Продумать архивирование.
