# FMDBx

[![Build Status](https://travis-ci.org/kohkimakimoto/FMDBx.svg?branch=master)](https://travis-ci.org/kohkimakimoto/FMDBx)

An extension of [FMDB](https://github.com/ccgus/fmdb) to provide ORM and migration functionality for your iOS application.

## Requirements

* iOS 7.0
* Xcode5
* ARC

> Note: I am testing this product on the above condition.

## Installation

You can install FMDBx via [CocoaPods](http://cocoapods.org).
Add the following line to your Podfile.

```
pod 'FMDBx'
```

## Usage

* [Database Manager](#database-manager)
* [Migration](#migration)
* [ORM](#orm)
* [Import data from CSV](#import-data-from-csv)

### Database Manager

Database Manager:`FMXDatabaseManager` class is a singleton instance that manages sqlite database files and FMDatabase instances connecting them.

#### Register a database

```Objective-C
[[FMXDatabaseManager sharedInstance] registerDefaultDatabaseWithPath:@"database.sqlite" migration:nil];
```

At the above example, you don't need to place `database.sqlite` file by hand. 
`FMXDatabaseManager` class automatically create initial empty `database.sqlite` file in the `NSDocumentDirectory`(Documents).

#### Get a FMDatabase instance from a registered database

```Objective-C
FMDatabase *db = [[FMXDatabaseManager sharedInstance] defaultDatabase];
[db open];

// your code for databse operations

[db close];
```

### Migration

`FMXDatabaseManager` can have a migration object to initialize and migrate database schema.

#### Register a database with a migration class

Create your migration class.

```Objective-C
@interface MyMigration : FMXDatabaseMigration

@end

@implementation MyMigration

- (void)migrate
{
    [self upToVersion:1 action:^(FMDatabase *db){
        [db executeUpdate:@""
         "create table users ("
         "  id integer primary key autoincrement,"
         "  name text not null,"
         "  age integer not null"
         ")"
         ];
    }];

    [self upToVersion:2 action:^(FMDatabase *db){
        // ... schema changes for version 2       
    }];

    [self upToVersion:3 action:^(FMDatabase *db){
        // ... schema changes for version 3       
    }];

    // ...etc
}

@end
```

Register database with an instance of migration class. It runs migration tasks.

```Objective-C
[[FMXDatabaseManager sharedInstance] registerDefaultDatabaseWithPath:@"database.sqlite" 
                                                           migration:[[MyMigration alloc] init]];
```

### ORM

It is designed as ActiveRecord.

#### Define a model class

You need to define model classes for each tables.
By default, a model class automatically maps a table which is named pluralized class name without prefix.(It's not strict. Just add `s` end of the term). 

For example, `ABCUser` model class maps `users` table at default.

```Objective-C
@interface ABCUser : FMXModel

@property (strong, nonatomic) NSNumber *id;
@property (strong, nonatomic) NSString *name;
@property (strong, nonatomic) NSNumber *age;

@end
```

You need to define `schema` method like the following to map each properties with table columns.

```Objective-C
@implementation ABCUser

- (void)schema:(FMXTableMap *)table
{
    [table hasIntIncrementsColumn:@"id"];   // defines as primary key.
    [table hasStringColumn:@"name"];
    [table hasIntColumn:@"age"];
}

@end
```

The model class needs primary key. So you need to define primary key configuration. Please see below example.

```Objective-C
[table hasIntIncrementsColumn:@"id"];

// or 

[table hasIntColumn:@"id" withPrimaryKey:YES];
```

If you want to change a mapped table name from a default,
you can specify table name like the following.

```Objective-C
@implementation ABCUser

- (void)schema:(FMXTableMap *)table
{
    [table setTableName:@"custom_users"];
}

@end
```

#### Insert, update and delete

You can use a model class to insert, update and delete data.

```Objective-C
ABCUser *user = [[ABCUser alloc] init];
user.name = @"Kohki Makimoto";
user.age = @(34);

// insert
[user save];

// update
user.age = @(44);
[user save];

// delete
[user delete];
```

#### Find by primary key

```Objective-C
ABCUser *user = (ABCUser *)[ABCUser modelByPrimaryKey:@(1)];
NSLog(@"Hello %@", user.name);
```

#### Find by where conditions

You can get a model.

```Objective-C
ABCUser *user = (ABCUser *)[ABCUser modelWhere:@"name = :name" parameters:@{@"name": @"Kohki Makimoto"}];
```

You can get multiple models.

```Objective-C
NSArray *users = [ABCUser modelsWhere:@"age = :age" parameters:@{@"age": @34}];
for (ABCUser *user in users) {
    NSLog(@"Hello %@!", user.name);
}
```

#### Count records by where conditions

```Objective-C
# Count all users.
NSInteger count = [ABCUser count];

# Count users whose name is 'Kohki Makimoto'.
NSInteger count = [ABCUser countWhere:@"name = :name" parameters:@{@"name": @"Kohki Makimoto"}];
```

### Import data from CSV 

You can add some data in your migration task or others.

```Objective-C
@interface MyMigration : FMXDatabaseMigration

@end

@implementation MyMigration

- (void)migrate
{
    [self upToVersion:1 action:^(FMDatabase *db){
        [db executeUpdate:@""
         "create table users ("
         "  id integer primary key autoincrement,"
         "  name text not null,"
         "  age integer not null"
         ")"
         ];

        [FMXCsvTable foreachFileName:@"users.csv" process:^(NSDictionary *row) {
            [ABCUser createWithValues:row database:db];
        }];
    }];
}

@end
```

CSV file is like the following. The header line is must.

```
id,name,age
1,Kohki Makimoto1,34
2,Kohki Makimoto2,35
3,Kohki Makimoto3,36
```

