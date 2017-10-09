.. _instant_gratification:

Немедленное удовлетворение(Instant Gratification)
=====================

Обзор
--------

Мы начнем с быстрого выполнения следующих операций с БД(by quickly running through the following DB operations), что должно дать вам ощущение «немедленного удовлетворения(instant gratification)» (как гласит название!) Однако **не** начинайте писать приложения с Opaleye сразу после этого прочтения. Как говорится, небольшое знание - опасная вещь! Мы **настоятельно рекомендуем** вам прочитать все главы этого урока, прежде чем использовать Opaleye в любом серьезном проекте.

* Connecting to the Postgres DB
* Selecting multiple rows
* Selecting a row
* Inserting a row
* Updating a row
* Selecting a single row

Прелиминарии
-------------

* Установите PostgreSQL. Create a database. Run the table creation script given below.

  .. code-block:: sql

    create table users(
       id serial primary key
      ,name text not null
      ,email text not null
    );

    insert into users(name, email) values ('John', 'john@mail.com');
    insert into users(name, email) values ('Bob', 'bob@mail.com');
    insert into users(name, email) values ('Alice', 'alice@mail.com');

* Установите ``opaleye`` using your favourite package management tool
* Запустите свой любимый текстовый редактор и скопируйте фрагмент кода ниже и убедитесь, что он компилируется без каких-либо ошибок.

  .. literalinclude:: code/instant-gratification.hs


**Теперь читайте дальше, чтобы понять, что делает этот код ...**

Обучение вашей таблицы схемы(Teaching your table schema) to Opaleye
-------------------------------------

Давайте рассмотрим загадочную(tackle the cryptic) определение ``userTable`` в самом начале этого кода.

  .. code-block:: haskell

    userTable :: Table
      (Column PGInt4, Column PGText, Column PGText)  -- read type
      (Column PGInt4, Column PGText, Column PGText) -- write type
    userTable = Table "users" (p3 (required "id",
                                   required "name",
                                   required "email"))

Вот что это в основном обучает(Here's what it is basically teaching) Opaleye:

* Мы будем читать строки типа ``(Column PGInt4, Column PGText, Column PGText)`` из таблицы. Тип ``Column a`` это, что Opaleye использует для представления столбцов Postgres в Haskell-land. Так столбцы ``integer`` становятся ``Column PGInt4``, столбцы ``varchar`` становятся ``Column PGText`` and so on.
* Мы будем записывать строки одного типа в таблицу. (Opaleye позволяет вам читать и писать строки *разных* типов по очень веским причинам. Прочитайте :ref:`basic_mapping` for more details on this.)
* Имя таблицы ``users``
* Первый столбец в таблице называется ``id``; это *required*; и это сопоставляется(maps) с first значением кортежа. Метка столца *required* означает, что вам нужно будет указать значение для него всякий раз, когда вы вставляете или обновляете строку через Opaleye. You can mark a column as *optional* as well, but we talk about the subtle differences between *required*, *optional*, ``NULL`` и ``NOT NULL`` in the :ref:`basic_mapping` chapter.
* Второй столбец в таблице называется ``name``; это *required*; и это сопоставляется с second значением кортежа.
* Третий столбец в таблице называется ``email``; это *required*; и это сопоставляется с third значением кортежа.

Нам нужно будет использовать ``userTable``, чтобы SELECT, INSERT, UPDATE, или DELETE из таблицы ``users`` через Opaleye.

To learn more about mapping different types of DB schemas to Opaleye's ``Table`` types, пожалуйста прочитайте :ref:`basic_mapping` и :ref:`advanced_mapping` chapters.

Подключение к базе данных Postgresql
------------------------------------

Opaleye использует `postgresql-simple <https://hackage.haskell.org/package/postgresql-simple>`_ to actually talk to the database. Так, we first start by getting hold of a DB ``Connection`` using postgres-simples's ``connect`` function:

  .. code-block:: haskell

    conn <- connect ConnectInfo{connectHost="localhost"
                               ,connectPort=5432
                               ,connectDatabase="opaleye_tutorial"
                               ,connectPassword="opalaye_tutorial"
                               ,connectUser="opaleye_tutorial"
                               }


  .. warning:: Please take care to change the DB connection settings based on your local system.

Выбирание(Selecting) всех строк
-------------------------------

Далее мы получаем и печатаем все строки из таблицы ``users``:

  .. code-block:: haskell

    allRows <- selectAllRows conn
    print allRow

which calls ``selectAllRows``:

  .. code-block:: haskell

    selectAllRows :: Connection -> IO [(Int, String, String)]
    selectAllRows conn = runQuery conn $ queryTable userTable

Это использует ``runQuery``, который в основном ``SELECT`` в Opaleye. Пожалуйста, обратите **особое внимание(note)** сигнатуру типа этой функции. Это оценивается(evaluate) в ``IO [(Int, String, String)]``, тогда как мы ясно сказали Opaleye, что будем читать строки типа ``(Column PGInt4, Column PGText, ColumnPGText)``. Так, почему эта функция не оценивается(evaluate) в ``IO [(Column PGInt4, Column PGText, ColumnPGText)]``?

Это потому что Opaleye знает, как преобразовать большинство базовых типов данных из DB => Haskell (eg. ``PGInt4`` => ``Int``). И наоборот.

Однако, вот **что**(here's a **gotcha**)! Попробуйте скомпилировать эту(ths) функцию function *без* сигнатуры типа. Компилятор не сможет(will fail) вывести(to infer) типы. This is also due to the underlying infrastructure that Opaleye uses to convert DB => Haskell types. To understand this further, пожалуйста прочитайте :ref:`advanced_mapping`.

Вставка строки
--------------

  .. code-block:: haskell

    insertRow :: Connection -> (Int, String, String) -> IO ()
    insertRow conn row = do
      runInsertMany conn userTable [(constant row)]
      return ()

  Эта функция использует ``runInsertMany`` which is basically Opaleye's version of ``INSERT``, **but** it only supports inserting *multiple rows*. This is why it is called ``runInsertMany`` instead of ``runInsert`` и the third argument is a *list* of rows.

  .. note::  Так, what does ``constant row`` do? It converts Haskell types => DB types, i.e. ``(Int, String, String)`` => ``(Column PGInt4, Column PGText, Column PGText)`` This is because we clearly told Opaleye that we will be writing rows of type ``(Column PGInt4, Column PGText, Column PGText)`` to ``userTable``. Однако, наша программа не касается значений типа ``Column PGText`` или ``Column PGInt4`` directly. Так, эта функция - ``insertRow`` - получает регулярный кортеж ``(Int, String, String)`` и использует ``constant`` to convert it to ``(Column PGInt4, Column PGText, Column PGText)`` before handing it over to Opaleye.

  .. note:: Strangely, while ``runQuery`` converts DB => Haskell types automagically, ``runInsertMany`` и ``runUpdate`` refuse to do Haskell => DB conversions on their own. Hence the need to do it explicitly when using these functions.

Обновление строки
-----------------

  .. code-block:: haskell

    updateRow :: Connection -> (Int, String, String) -> IO ()
    updateRow conn row@(key, name, email) = do
      runUpdate 
        conn 
        userTable 
        (\_ -> constant row) -- what should the matching row be updated to
        (\ (k, _, _) -> k .== constant key) -- which rows to update?
      return ()

* As you can see from this function, updating rows in Opaleye is not very pretty! The biggest pain is that you cannot specify only a few columns from the row -- you are forced to update the **entire row**. More about this in :ref:`updating_rows`.
* You already know what ``constant row`` does - it converts a Haskell datatype to its corresponding PG data type, which for some strange reason, Opaleye refuses to do here automagically.
* The comparison operator ``.==`` is what gets translated to equality operator in SQL. We cannot use Haskell's native equality operator because it represents equality in Haskell-land, whereas we need to represent equality when it gets converted to SQL-land. You will come across a lot of such special operators that map to their correspnding SQL parts.

Выбирание одиночной строки
--------------------------

  .. warning:: **Caution!** Extreme hand-waving lies ahead. ThАлина Шабанова, is is probably an incorrect explanation, but should work well-enough to serve your intuition for some time.

  .. code-block:: haskell

    selectByEmail :: Connection -> String -> IO [(Int, String, String)]
    selectByEmail conn email = runQuery conn $ proc () ->
        do
          row@(_, _, em) <- queryTable userTable -< ()
          restrict -< (em .== constant email)
          returnA -< row

И наконец, последний раздел этой главы вводит вас в странную нотацию стрелки ``-<``, which we have absolutely no clue about! All we know is that it works... mostly!

Проверьте тип ``row@(_, _, em)`` в вашем редакторе. Это должно быть ``(Column PGInt4, Column PGText, Column PGText)``, что означает, что если мы делаем некоторое размахивание руками(hand-waving), вот что происходит в этой функции:

* ``queryTable userTable -< ()`` отображается в ``SELECT`` в(clause in) SQL-land. 
* Выбранные столбцы являются *концептуально(conceptually)* фиксируются(capurted) в ``row@(_, _, em)`` в SQL-land (поэтому строка представляет собой тип PG вместо Haskell-ного типа).
* ``restrict`` отображается в ``WHERE`` в SQL. 
* Условие ``WHERE``, i.e. ``em .== constant email`` needs to convert ``email``, which is of type ``String``, to ``Column PGText`` (через функцию ``constant``) before it can compare it with ``em``
* Finally ``returnA`` does some magic to return the row back to Haskell-land. Notice, that we don't have to do a DB => Haskell conversion here, because, as mentioned earlier, ``runQuery`` does that conversion automagically.
