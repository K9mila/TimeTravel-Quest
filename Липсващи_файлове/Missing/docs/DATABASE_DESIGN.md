# Проект на базата данни
Това е логически модел за Седмица 1; изпълнимият schema.sql и проверените RLS политики се разработват в Седмица 2. Визуалната схема е ER_DIAGRAM.svg; Mermaid версията е ER_DIAGRAM.mmd.

## Полета и ограничения
Всички полета са NOT NULL, освен изрично посочените като nullable. UUID ключовете получават генерирана стойност, освен profiles.id, който идва от Auth. created_at/updated_at са timestamptz; сървърът задава времето. Посочените връзки се индексират.

| Таблица | Полета | Ключове и ограничения |
|---|---|---|
| profiles | id uuid; display_name text; created_at timestamptz | PK id, FK → auth.users.id ON DELETE CASCADE; име 2–30 символа; не пазим пароли или email тук |
| eras | id uuid; title text; sort_order smallint; intro text; character_name text; character_dialogue text | PK id; UNIQUE sort_order; CHECK sort_order BETWEEN 1 AND 5 |
| events | id uuid; era_id uuid; title text; year integer; source_url text | PK id; FK era_id → eras.id; CHECK year != 0 |
| artifacts | id uuid; era_id uuid; name text; description text; source_url text; image_path text nullable | PK id; FK era_id → eras.id; UNIQUE era_id за една ключова награда на глава |
| quests | id uuid; era_id uuid; prompt text; type text; difficulty smallint; sort_order smallint; options jsonb; correct_answer jsonb; hint text; explanation text; source_url text; verified boolean | PK id; FK era_id → eras.id; UNIQUE(era_id,sort_order); CHECK difficulty BETWEEN 1 AND 3; type IN ('choice','chronology'); verified DEFAULT false |
| player_progress | user_id uuid; era_id uuid; status text; difficulty smallint; success_streak smallint; error_streak smallint; updated_at timestamptz | PK(user_id,era_id); FK user_id → profiles.id ON DELETE CASCADE; FK era_id → eras.id; status IN ('in_progress','completed'); difficulty 1–3; поредици >= 0 |
| quest_progress | user_id uuid; quest_id uuid; attempts integer; solved boolean; first_correct boolean nullable; hint_used boolean; updated_at timestamptz | PK(user_id,quest_id); FK user_id → profiles.id ON DELETE CASCADE; FK quest_id → quests.id; attempts >= 0; nullable first_correct преди първи отговор |
| inventory | user_id uuid; artifact_id uuid; acquired_at timestamptz | PK(user_id,artifact_id); FK user_id → profiles.id ON DELETE CASCADE; FK artifact_id → artifacts.id |

Персонажът е един на епоха и е описан в eras. Отделна таблица characters е нужна само ако добавим няколко персонажа на сцена. quest_progress е добавена към предложените в заданието таблици, за да се пазят решените задачи и да не се дават повторни награди. Финалната хронология се извежда от events; не е необходима дублирана таблица timeline.

options при choice съдържа масив от обекти {id,text}; correct_answer съдържа правилното option ID. При chronology options съдържа event IDs, а редът се проверява по events.year. JSON структурата се проверява в приложението и при импорта на съдържание. Липсващ FK вътре в JSON е ограничение: seed валидаторът трябва да проверява всички event IDs.

## Достъп и RLS
Единствената приложна роля в MVP е регистриран играч. Учителят може да разгледа проекта с демо профил. Съдържанието се въвежда от разработчиците през контролирани миграции, а не чрез публичен администраторски екран.

| Данни | Четене | Запис |
|---|---|---|
| eras, events, artifacts | Регистрирани играчи | Само контролирана миграция/администратор на базата |
| quests | Регистрирани играчи; само verified=true | Само контролирана миграция/администратор на базата |
| profiles | Само auth.uid() = id | Собствен профил; id не се променя |
| player_progress, quest_progress, inventory | Само auth.uid() = user_id | Само през контролирани функции за отговор, награда и нулиране |

RLS се включва за всяка публична таблица. Личните SELECT политики използват USING(auth.uid() = user_id); при profiles сравняват с id. При собствен INSERT/UPDATE на profiles се използва и WITH CHECK. За таблиците с игрови прогрес се отнемат директните клиентски права за промяна; функциите вземат потребителя от auth.uid(), а не от подаден user_id.

Функцията submit_answer проверява отключването и задачата, записва опита и актуализира адаптацията. При четири решени задачи завършва епохата и добавя наградата в една транзакция. Заключване на съответния ред и UNIQUE ключовете предотвратяват дублиране при двойно натискане. Контролираната функция reset_progress изтрива само личните записи след потвърждение в UI. SECURITY DEFINER функции изискват фиксиран search_path, ограничен EXECUTE и проверка за автентикация; проверяват се в седмица 2–3.

В тази учебна версия отговорите са достъпни за инспектиране от регистрирания клиент, тъй като няма състезателна класация. RLS пази чуждите лични данни, а не прави въпросите тайни. Ако по-късно добавим състезание, отговорите трябва да се отделят в частна таблица и проверката да остане само на сървъра.

## Връзки и изтриване
Епоха има много събития и задачи, и най-много един ключов артефакт. Играчът има много записи за епохи и задачи. inventory свързва играчи и артефакти много-към-много.

Изтриването на auth потребител премахва личните данни чрез каскада. Историческо съдържание с използвани връзки се пази с RESTRICT; не се изтрива по време на игра. Заключването на главите се извежда от sort_order и завършената предишна глава, вместо да се пази второ независимо поле locked.

## Приемане на сигурността
С два различни тестови профила се доказва, че A не чете/променя данните на B. Невлязъл клиент няма личен достъп. Директен INSERT на награда и директна промяна на completed се отказват. Публичният клиент използва само предвидения за него Supabase ключ; service-role ключ никога не влиза в GitHub или браузъра.