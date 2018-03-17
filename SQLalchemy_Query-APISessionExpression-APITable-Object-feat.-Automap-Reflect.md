# [SQLalchemy] Query API(ORM)/Expression API(Core)簡介 feat. Automap, Reflect
Cheatsheet - [https://www.pythonsheets.com/notes/python-sqlalchemy.html](https://www.pythonsheets.com/notes/python-sqlalchemy.html)


----------
<font color="red">**Concept**</font>

SQLAlchemy提供了三種方式表示資料庫中表的結構：
1. <font color="blue">Core</font> - Table Object 給 SQLalchemy 的 Expression API 使用
2. <font color="blue">ORM</font> - Declarative Class 給 SQLalchemy 的 Session 使用
3. <font color="blue">Reflect/Automap</font>: 分別自動映射資料庫表的結構回上面兩種


sqlalchemy提供了一種資料庫<font color="blue">自動映射</font>的方式將資料庫的schema映射成sqlalchemy可以使用的object，
這種概念可以分成兩種層級：Declarative Class、Table Object

<font color="red">**Declarative Class**</font>

declarative class - 執行session query的基本單位，<font color="blue">**必須有primary key欄位**</font>
Ex.

```python
    from sqlalchemy import Column, Integer, String
    from sqlalchemy.ext.declarative import declarative_base
    
    Base = declarative_base()
    
    class SomeClass(Base):
        __tablename__ = 'some_table'
        id = Column(Integer, primary_key=True)
        name =  Column(String(50))
```

Reference:

1. [declarative class](http://docs.sqlalchemy.org/en/latest/orm/extensions/declarative/basic_use.html#basic-use)
2. [Primary key required](http://docs.sqlalchemy.org/en/latest/orm/session_basics.html#what-does-the-session-do) (自行搜尋primary)
3. [Session Query API](http://docs.sqlalchemy.org/en/latest/orm/query.html#query-api)



- 自動映射方式 - 使用**automap**自動映射
  關鍵字：
    automap_base()、prepare()

Ex.
```python
    from sqlalchemy.ext.automap import automap_base
    from sqlalchemy.orm import Session
    from sqlalchemy import create_engine
    
    Base = automap_base()
    
    # engine, suppose it has two tables 'user' and 'address' set up
    engine = create_engine("sqlite:///mydatabase.db")
    
    # reflect the tables
    Base.prepare(engine, reflect=True)
    
    # mapped classes are now created with names by default
    # matching that of the table name.
    User = Base.classes.user
    Address = Base.classes.address
    
    session = Session(engine)
    
    # rudimentary relationships are produced
    session.add(Address(email_address="foo@bar.com", user=User(name="foo")))
    session.commit()
    
    # collection-based relationships are by default named
    # "<classname>_collection"
    print (u1.address_collection)
```

Reference:

1. [Automap Basic Use](http://docs.sqlalchemy.org/en/latest/orm/extensions/automap.html#basic-use)


<font color="red">**Table Object**</font>

table object - 執行expression api的基本單位，<font color="blue">**不需**primary key欄位</font>
Ex.

```python
    from sqlalchemy import table, column
    
    user = table("user",
            column("id"),
            column("name"),
            column("description"),
    )
```

Reference:

1. [Table Object](http://docs.sqlalchemy.org/en/latest/core/selectable.html#sqlalchemy.sql.expression.TableClause)
2. [Expression API](http://docs.sqlalchemy.org/en/latest/core/tutorial.html#sql-expression-language-tutorial)


- 自動映射方式 - 使用**reflect**自動映射
  關鍵字：
    reflect()

Ex.

```python
    # Reflecting All Tables at Once
    meta = MetaData()
    meta.reflect(bind=someengine)
    users_table = meta.tables['users']
    addresses_table = meta.tables['addresses']
```

Reference:

1. [reflect all tables at once](http://docs.sqlalchemy.org/en/latest/orm/extensions/automap.html#basic-use)
2. [Document: Reflect](http://docs.sqlalchemy.org/en/latest/core/metadata.html#sqlalchemy.schema.MetaData.reflect)


<font color="red">**Table Object → Declarative Class**</font>

table object 跟 declarative class的關係可以想像成一般型跟進化型的概念，table object經過包裝可以升級成 declarative class，同理也可以從 declarative class 中的property中取得 table object。
Ex.

```python
    class MyClass(Base):
        __table__ = Table('my_table', Base.metadata,
            Column('id', Integer, primary_key=True),
            Column('name', String(50))
        )
```

其中 declarative class 中的 __table__ 屬性就是 table object本身。
Reference:

1. [Create a declarative class from table object](http://docs.sqlalchemy.org/en/latest/orm/extensions/declarative/table_config.html#using-a-hybrid-approach-with-table)


<font color="red">**Automap/Reflect DB View**</font>

上述的兩種自動映射方式都只有處理一般已經存在db中的table，如果想要針對<font color="blue">db的view也使用automap</font>的話要使用下述的方式進行處理：

概念：


基本上<font color="blue">Automap不支援DB View自動映射</font>，所以只能使用table object的方式reflect映射View。
所以本質上自動映射的View只有支援到table object層級，也就是說只能使用Expression API。但是上面的section也有提到，table object是可以包裝成declarative class的，所以如果能自行包裝成declarative class這樣當然也就能夠使用Session Query API囉！

步驟參考：

1. [Reflect View to table object](http://docs.sqlalchemy.org/en/latest/core/reflection.html#reflecting-views) - 

[範例](https://stackoverflow.com/questions/20518521/is-possible-to-mapping-view-with-class-using-mapper-in-sqlalchemy)
Ex.

```python
      view = Table( 'viewname', 
                    metadata,
                    autoload=True, 
                    autoload_with=engine)
```

  其中如果metadata已經bind engine的話，參數中就不用使用autoload_with了！
  在這個範例中如果沒有綁定engine的話會出現如下的問題：
    <h6>sqlalchemy.exc.UnboundExecutionError: No engine is bound to this Table's MetaData. Pass an engine to the Table via autoload_with=&lt;someengine>, or associate the MetaData with an engine via metadata.bind=&lt;someengine></h6>

  [Error Reference](https://stackoverflow.com/questions/7802981/using-sqlalchemy-declarative-base-and-autoload-true-in-pyramid)


2. Table Object → Declarative Class

```python
    class MyViewClass(Base):
        __table__ = Table( 'viewname', 
                            metadata, 
                            Column('id',Integer, primary_key=True), 
                            autoload=True, 
                            autoload_with=engine,
                            extend_existing=True )
```

這邊主要要注意下面兩點

  1. 可能需override新增一個定義的primary_key
    有很多的時候db view是不會有primary key的，而如第一段定義所述，declarative class必要一個primary_key，所以這邊必須自己手動新增一個column
    [Reference](http://docs.sqlalchemy.org/en/latest/faq/ormconfiguration.html#how-do-i-map-a-table-that-has-no-primary-key)


  2. 增加extend_existing這個參數
    如果綁定engine的方式不是使用autoload_with=engine的話，也可以在Base.metadata.bind中直接綁定engine，但這種狀況下可能因為你automap declarative class後直接存在Base這個global變數，直接導致除了第一次request可以成功automap，之後的request會直接噴錯，出現如下錯誤：
    <h6>sqlalchemy.exc.InvalidRequestError: Table 'search_engine_goods' is already defined for this MetaData instance.  Specify 'extend_existing=True' to redefine options and columns on an existing Table object.</h6>
    這時候只要多加一個 extend_existing=True即可，extend_existing的主要用途就是在允許declarative class可以複寫已經存在的declarative物件（如果程式架構有規劃好可能有機會斃掉這個問題）。
    

Reference:

1. [Base.metadata.bind](https://stackoverflow.com/questions/4526498/sqlalchemy-declarative-syntax-with-autoload-reflection-in-pylons/4555998#4555998)
2. [extend_existing](http://docs.sqlalchemy.org/en/latest/core/metadata.html#sqlalchemy.schema.Table.params.extend_existing)




