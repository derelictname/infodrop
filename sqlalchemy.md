# sqlalchemy

Позволяет описать таблицы в виде классов и взаимодействовать с записями, как с экземлярами класса.
В версии 2.0 произошли изменения, некоторые из которых доступны в 1.4.

- [Quick Start](https://docs.sqlalchemy.org/en/14/orm/quickstart.html)
- [Tutorial](https://docs.sqlalchemy.org/en/14/tutorial/index.html)

    pip install --user "sqlalchemy>=1.4"

Textual SQL - самый нижний уровень работы с базой данных

```python
from sqlalchemy import create_engine, text

RDBMS_URL = 'mysql://user1:Pa2s4sw5or4d!@localhost/db1?charset=utf8mb4'
engine = create_engine(RDBMS_URL, echo=True, future=True)

with engine.connect() as conn:
    sql = text('SELECT name, description FROM games')
    result = conn.execute(sql).all()
    for row in result:
        print(row.name, row.description)
```

## ORM

Описание базы данных хранится в объекте класса MetaData, который мы наполняем определениями таблиц класса Table. Структуру таблиц можно описать самостоятельно (объявление) или отзеркалить из реальной базы данных (отражение). При использовании ORM объект MetaData хранится в объекте Registry.

Вместо ручного наполнения MetaData объектами Table, есть более простой декларативный способ

```python
from sqlalchemy import create_engine, Column, Integer, String, Text, ForeignKey, select
from sqlalchemy.orm import declarative_base, relationship, Session

# Базовый класс для создания своих маппингов
Base = declarative_base()

# Можно создать свой родительский класс, если в таблицах есть одинаковые столбцы
class CommonBase(Base):
    # Атрибут, указывающий, что это НЕ маппинг (не создаёт метаданных)
    __abstract__ = True

    id = Column(Integer, primary_key=True)

    def __repr__(self):
        return f'{self.__class__.__name__} id={self.id!r}'

# Если нам необходимо только чтение, то все столбцы необязательно указывать
class Game(CommonBase):
    __tablename__ = 'games'
    genreid = Column(Integer, ForeignKey('genres.id'))
    title = Column(String(30), nullable=False, unique=True)
    description = Column(Text)
    genre = relationship('Genre', back_populates='games')

class Genre(CommonBase):
    __tablename__ = 'genres'
    name = Column(String(20), nullable=False, unique=True)
    games = relationship('Game', back_populates='genre')


# Проверим созданные маппинги
print(Base.metadata.tables)

# Адреса для соединения с СУБД
RDBMS_URL1 = 'sqlite://'
RDBMS_URL2 = 'mysql://user1:Pa2s4sw5or4d!@localhost/db1?charset=utf8mb4'
engine = create_engine(RDBMS_URL1, echo=True, future=True)

# По определению маппингов можно создать отсутствующие таблицы
# Если таблица уже есть, то она не будет затронута
Base.metadata.create_all(engine)
```

Один из способов работы с БД - через объект сессии, оперирующий объектами, связанными со своей записью в БД

```python
with Session(engine) as session:

    # INSERT

    # Объекты записей являются экземпляром класса таблицы
    action = Genre(name='Action')
    unreal = Game(title='Unreal', genre=action, description='Skaarj can dodge')
    session.add_all([
        unreal,
        Game(title='Half Life', genre=action, description='First day in Black Mesa')
    ])
    session.add(Game(title='Starcraft', genre=Genre(name='Strategy')))

    # UPDATE

    unreal.title = 'Unreal Tournament'

    # Создаём объект записи через запрос в БД
    sql = select(Game).where(Game.title == 'Half Life')
    halflife = session.scalars(sql).one()
    halflife.title = 'Half Life: Opposing Force'

    # DELETE

    session.delete(unreal)

    # session.add(), session.delete() и изменение объектов фиксирует изменение в сессии.
    # Реальный SQL запрос будет выполнен лишь при запросе изменённых данных или
    # при вызове session.flush() (выполнится INSERT/UPDATE/DELETE) или
    # session.commit() (выполнится INSERT/UPDATE/DELETE и потом COMMIT).
    # Если не будет выполнена фиксация изменений в базе данных с помощью session.commit(),
    # то при вызове session.rollback() или закрытий сессии будет выполнен ROLLBACK.
    session.commit()

    # SELECT

    sql = select(Game).join(Genre)
    # метод scalars возвращает объект ScalarResult
    # метод execute возвращает объект Row
    print(session.scalars(sql).all())

```
