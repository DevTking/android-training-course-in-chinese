# Saving to database保存到数据库

> 编写:[kesenhoo](https://github.com/kesenhoo) - 原文:<http://developer.android.com/training/basics/data-storage/database.html>

对于重复或者结构化的数据（如联系人信息）等保存到DB是个不错的主意。这节课假定你已经熟悉SQL数据库的操作。在Android上可能会使用到的APIs，可以从[android.database.sqlite](http://developer.android.com/reference/android/database/sqlite/package-summary.html)包中找到。

## Define a Schema and Contract[定义Schema与Contract]
SQL中一个中重要的概念是schema：一种DB结构的正式声明。schema是从你创建DB的SQL语句中生成的。你可能会发现创建一个伴随类（companion class）是很有益的，这个类成为合约类（contract class）,它用一种系统化并且自动生成文档的方式，显示指定了你的schema样式。

Contract Clsss是一些常量的容器。它定义了例如URIs, 表名，列名等。这个contract类允许你在同一个包下与其他类使用同样的常量。 它让你只需要在一个地方修改列名，然后这个列名就可以自动传递给你整个code。

一个组织你的contract类的好方法是在你的类的根层级定义一些全局变量，然后为每一个table来创建内部类。

**Note:**通过实现 [BaseColumns](http://developer.android.com/reference/android/provider/BaseColumns.html) 的接口，你的内部类可以继承到一个名为_ID的主键，这个对于Android里面的一些类似cursor adaptor类是很有必要的。这样能够使得你的DB与Android的framework能够很好的相容。

例如，下面的例子定义了表名与这个表的列名：

```java
public static abstract class FeedEntry implements BaseColumns {
    public static final String TABLE_NAME = "entry";
    public static final String COLUMN_NAME_ENTRY_ID = "entryid";
    public static final String COLUMN_NAME_TITLE = "title";
    public static final String COLUMN_NAME_SUBTITLE = "subtitle";
    ...
}
```

为了防止一些人不小心实例化contract类，像下面一样给一个空的构造器。

```java
// Prevents the FeedReaderContract class from being instantiated.
private FeedReaderContract() {}
```

## Create a Database Using a SQL Helper[使用SQL Helper创建DB]
当你定义好了你的DB应该是什么样之后，你应该实现那些创建与维护db与table的方法。下面是一些典型的创建与删除table的语句。

```java
private static final String TEXT_TYPE = " TEXT";
private static final String COMMA_SEP = ",";
private static final String SQL_CREATE_ENTRIES =
    "CREATE TABLE " + FeedReaderContract.FeedEntry.TABLE_NAME + " (" +
    FeedReaderContract.FeedEntry._ID + " INTEGER PRIMARY KEY," +
    FeedReaderContract.FeedEntry.COLUMN_NAME_ENTRY_ID + TEXT_TYPE + COMMA_SEP +
    FeedReaderContract.FeedEntry.COLUMN_NAME_TITLE + TEXT_TYPE + COMMA_SEP +
    ... // Any other options for the CREATE command
    " )";

private static final String SQL_DELETE_ENTRIES =
    "DROP TABLE IF EXISTS " + TABLE_NAME_ENTRIES;
```

就像保存文件到设备的 internal storage 一样，Android会保存db到你的程序的private的空间上。你的数据是受保护的，因为那些区域默认是私有的，不可被其他程序所访问。

在[SQLiteOpenHelper](http://developer.android.com/reference/android/database/sqlite/SQLiteOpenHelper.html)类中有一些很有用的APIs。当你使用这个类来做一些与你的db有关的操作时，系统会对那些有可能比较耗时的操作（例如创建与更新等）在真正需要的时候才去执行，而不是在app刚启动的时候就去做那些动作。你所需要做的仅仅是执行 getWritableDatabase() 或者 getReadableDatabase().

**Note:**因为那些操作可能是很耗时的，请确保你在background thread（AsyncTask or IntentService）里面去执行 getWritableDatabase() 或者 getReadableDatabase() 。

为了使用 SQLiteOpenHelper, 你需要创建一个子类并重写 onCreate(), onUpgrade() 与 onOpen()等callback方法。你也许还需要实现 onDowngrade(), 但是这并不是必需的。

例如，下面是一个实现了SQLiteOpenHelper 类的例子：

```java
public class FeedReaderDbHelper extends SQLiteOpenHelper {
    // If you change the database schema, you must increment the database version.
    public static final int DATABASE_VERSION = 1;
    public static final String DATABASE_NAME = "FeedReader.db";

    public FeedReaderDbHelper(Context context) {
        super(context, DATABASE_NAME, null, DATABASE_VERSION);
    }
    public void onCreate(SQLiteDatabase db) {
        db.execSQL(SQL_CREATE_ENTRIES);
    }
    public void onUpgrade(SQLiteDatabase db, int oldVersion, int newVersion) {
        // This database is only a cache for online data, so its upgrade policy is
        // to simply to discard the data and start over
        db.execSQL(SQL_DELETE_ENTRIES);
        onCreate(db);
    }
    public void onDowngrade(SQLiteDatabase db, int oldVersion, int newVersion) {
        onUpgrade(db, oldVersion, newVersion);
    }
}
```

为了访问你的db，需要实例化你的 SQLiteOpenHelper的子类：

```java
FeedReaderDbHelper mDbHelper = new FeedReaderDbHelper(getContext());
```

## Put Information into a Database [添加信息到DB]
通过传递一个 ContentValues 对象到 insert() 方法：

```java
// Gets the data repository in write mode
SQLiteDatabase db = mDbHelper.getWritableDatabase();

// Create a new map of values, where column names are the keys
ContentValues values = new ContentValues();
values.put(FeedReaderContract.FeedEntry.COLUMN_NAME_ENTRY_ID, id);
values.put(FeedReaderContract.FeedEntry.COLUMN_NAME_TITLE, title);
values.put(FeedReaderContract.FeedEntry.COLUMN_NAME_CONTENT, content);

// Insert the new row, returning the primary key value of the new row
long newRowId;
newRowId = db.insert(
         FeedReaderContract.FeedEntry.TABLE_NAME,
         FeedReaderContract.FeedEntry.COLUMN_NAME_NULLABLE,
         values);
```

insert() 方法的第一个参数是table名，第二个参数会使得系统自动对那些 ContentValues 没有提供数据的列填充数据为null，如果第二个参数传递的是null，那么系统则不会对那些没有提供数据的列进行填充。

## Read Information from a Database [从DB中读取信息]
为了从DB中读取数据，需要使用 query() 方法， 传递你需要查询的条件。查询后会返回一个 Cursor 对象。

```java
SQLiteDatabase db = mDbHelper.getReadableDatabase();

// Define a projection that specifies which columns from the database
// you will actually use after this query.
String[] projection = {
    FeedReaderContract.FeedEntry._ID,
    FeedReaderContract.FeedEntry.COLUMN_NAME_TITLE,
    FeedReaderContract.FeedEntry.COLUMN_NAME_UPDATED,
    ...
    };

// How you want the results sorted in the resulting Cursor
String sortOrder =
    FeedReaderContract.FeedEntry.COLUMN_NAME_UPDATED + " DESC";

Cursor c = db.query(
    FeedReaderContract.FeedEntry.TABLE_NAME,  // The table to query
    projection,                               // The columns to return
    selection,                                // The columns for the WHERE clause
    selectionArgs,                            // The values for the WHERE clause
    null,                                     // don't group the rows
    null,                                     // don't filter by row groups
    sortOrder                                 // The sort order
    );
```

下面是演示如何从course对象中读取数据信息：

```java
cursor.moveToFirst();
long itemId = cursor.getLong(
    cursor.getColumnIndexOrThrow(FeedReaderContract.FeedEntry._ID)
);
```

## Delete Information from a Database [删除DB中的信息]
和查询信息一样，删除数据，同样需要提供一些删除标准。DB的API提供了一个防止SQL注入的机制来创建查询与删除标准。
**SQL Injection**(*随着B/S模式应用开发的发展，使用这种模式编写应用程序的程序员也越来越多。但是由于程序员的水平及经验也参差不齐，相当大一部分程序员在编写代码的时候，没有对用户输入数据的合法性进行判断，使应用程序存在安全隐患。用户可以提交一段数据库查询代码，根据程序返回的结果，获得某些他想得知的数据，这就是所谓的SQL Injection，即SQL注入*)

这个机制把查询语句划分为选项条款与选项参数两部分。条款部分定义了查询的列是怎么样的，参数部分用来测试是否符合前面的条款。(*这里翻译的怪怪的，附上原文，The clause defines the columns to look at, and also allows you to combine column tests. The arguments are values to test against that are bound into the clause.*) 因为处理的结果与通常的SQL语句不同，这样可以避免SQL注入问题。

```java
// Define 'where' part of query.
String selection = FeedReaderContract.FeedEntry.COLUMN_NAME_ENTRY_ID + " LIKE ?";
// Specify arguments in placeholder order.
String[] selelectionArgs = { String.valueOf(rowId) };
// Issue SQL statement.
db.delete(table_name, mySelection, selectionArgs);
```

## Update a Database [更新数据]
当你需要修改DB中的某些数据时，使用 update() 方法。

更新操作结合了插入与删除的语法。

```java
SQLiteDatabase db = mDbHelper.getReadableDatabase();

// New value for one column
ContentValues values = new ContentValues();
values.put(FeedReaderContract.FeedEntry.COLUMN_NAME_TITLE, title);

// Which row to update, based on the ID
String selection = FeedReaderContract.FeedEntry.COLUMN_NAME_ENTRY_ID + " LIKE ?";
String[] selelectionArgs = { String.valueOf(rowId) };

int count = db.update(
    FeedReaderDbHelper.FeedEntry.TABLE_NAME,
    values,
    selection,
    selectionArgs);
```
