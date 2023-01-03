# Steps to Normalize Tables

## Identify the item_type

## See what properties are available for that item_type

```
select meta_key, count(*)
from tc_meta
where item_type = 'user'
group by meta_key
```

```
meta_key  count
email 430
status  430
fullname  429
role  420
password  311
desk  113
facebook_test_date  3
facebook_test_date_searches 3
facebook_test_date_high_profiles  1
facebook_test_date_med_profiles 1
facebook_test_date_other_profiles 1
```

Looks like all users have an email and status, so those columns should be not nullable. Those facebook columns look useless, so lets ignore those. We should avoid storing passwords in our db for now because authentication happens through their api. The rest (fullname, role, desk) should be nullable columns


## Write a query that will produce the table

*Write the query*
Select from the `tc_item` table where `item_type` = 'user'. Then for each property, join the `tc_meta` table where the `item_id` matches and the `meta_key` equals the name of the property.

An important note is to use `left join` if you are getting a property that can be null. Otherwise your query will not return users that don't have a fullname but still need to be in the users table!

*How do I know what column each property value is in?*
```
select *
from tc_meta
where item_type = 'user'
```

The output shows that email is found in the `meta_string` column and role is found in the `meta_item_id` column.

*Example*
Note there is a bug in this, which we will discuss in the next section.
```
select
  tc_items.item_id as id,
  tc_items.item_creation as created_at,
  tc_items.item_creator as created_by_id,
  email.meta_string as email,
  status.meta_item_id as status_id,
  fullname.meta_string as fullname,
  role.meta_item_id as role_id
  desk.meta_item_id as desk_id
from tc_items
  join tc_meta email on email.item_id = tc_items.item_id and email.meta_key = 'email'
  join tc_meta status on status.item_id = tc_items.item_id and status.meta_key = 'status'
  left join tc_meta fullname on fullname.item_id = tc_items.item_id and fullname.meta_key = 'fullname'
  left join tc_meta role on role.item_id = tc_items.item_id and role.meta_key = 'role'
  left join tc_meta desk on desk.item_id = tc_items.item_id and desk.meta_key = 'desk'
where tc_items.item_type = 'user'
```

## Make sure the numbers add up
Its important the numbers add up because this query will be used to INSERT rows in the new users table.

If you wrap your query in a `select count(*) from (your query here ) as foo` it should equal the total number of items for your item_type. For example `select count(*) from tc_items where item_type = 'user';` returns 430 so there should be 430 rows from your query.

Reasons the numbers might not add up:
- You are using `join` but should really be using `left join` for a property that is nullable
- You ran into a m2m relationship


### Aha! Our numbers don't add up
One of your joins is a m2m relationship. For example I got 512 users (instead of 430) when I ran the following and it turned out that a user can have multiple desks.

```
select
  count(*)
from (
  select
    tc_items.item_id as id,
    tc_items.item_creation as created_at,
    tc_items.item_creator as created_by_id,
    email.meta_string as email,
    status.meta_item_id as status_id,
    fullname.meta_string as fullname,
    role.meta_item_id as role_id,
    desk.meta_item_id as desk_id
  from tc_items
    join tc_meta email on email.item_id = tc_items.item_id and email.meta_key = 'email'
    join tc_meta status on status.item_id = tc_items.item_id and status.meta_key = 'status'
    left join tc_meta fullname on fullname.item_id = tc_items.item_id and fullname.meta_key = 'fullname'
    left join tc_meta role on role.item_id = tc_items.item_id and role.meta_key = 'role'
    left join tc_meta desk on desk.item_id = tc_items.item_id and desk.meta_key = 'desk'
  where tc_items.item_type = 'user'
  order by id desc
) as _
;
```

What do we do?!? We need to get this query to return exactlly 430 rows if we want the same number of users in the new table. This will bring us to our next section on relationships

## Handling Relationships
There are a few different relationships a user has. A user can only have one status and one role but can have many desks. Also because desks can belong to many users, this is a m2m relationship. When this happens we need to define a new table to hold the m2m relationship (user_id, desk_id).

Its also possible to spot m2m relationships when creating

```
select meta_key, count(*)
from tc_meta
where item_type = 'company_user'
group by meta_key;
```

## Create the table

*What should the table name*
The table name _must_ be identical to `item_type`. Do not pluralize the table name. If the `item_type` is `user` then the table name should be `user`. We can pluralize once the old tables are obsolete.

*Should the column be unique?*
Use common sense when deciding if a column should be unique. And use the following query to check if the column _can_ be unique. In this example we are checking if user email can be unique. You would think yes but this actually shows there are 6 users with email `----@tamcsolutions.info`. So email cannot be a unique column, which is a bummer. 
```
select tc_meta.meta_string, count(*)
from tc_items
  join tc_meta on tc_meta.item_id = tc_items.item_id
where tc_items.item_type = 'user' and tc_meta.meta_key = 'email'
group by tc_meta.meta_string
having count(*) > 1;
```

*What type should each column be?*
Look at the table definition for tc_meta. It will tell you what type each value column (e.g. meta_int, meta_string, meta_date, etc.) will be. For example email is defined in the `meta_string` column and the type for that column is `varchar(255)`, which means the email column on the new user table should be `varchar(255)` as well.

*When should I add an index?*
I won't cover that here, but feel free to add an index for a performance bost.

*Example*
```
create table if not exists user (
  id int primary key,
  created_at datetime default CURRENT_TIMESTAMP not null,
  created_by_id int not null,
  email varchar(255) not null,
  status_id int not null,
  full_name varchar(255),
  role_id int
) character set = utf8;
```

*Update `tamc_tables.sql`*
Add your `create table` statement to `tamc_tables.sql`. Its important for this file to represent the statements required to create each table. Its okay if they change, there are no need for migrations at this stage because we can just destroy can re-create each table within minutes.

## Create the data migration
The data migration at this point is really easy. Assuming the column positioning of your query matches up with the column positioning of your `create table` statement, you can just throw `insert into user` before your big query and it will just populate the entire user table in one shot.

*Example*
```
insert into user
select
  tc_items.item_id as id,
  tc_items.item_creation as created_at,
  tc_items.item_creator as created_by_id,
  email.meta_string as email,
  status.meta_item_id as status_id,
  fullname.meta_string as fullname,
  role.meta_item_id as role_id
from tc_items
  join tc_meta email on email.item_id = tc_items.item_id and email.meta_key = 'email'
  join tc_meta status on status.item_id = tc_items.item_id and status.meta_key = 'status'
  left join tc_meta fullname on fullname.item_id = tc_items.item_id and fullname.meta_key = 'fullname'
  left join tc_meta role on role.item_id = tc_items.item_id and role.meta_key = 'role'
where tc_items.item_type = 'user'
```

*Update `data_migrations.sql`*
Add your `insert` statement to `data_migrations.sql`

*Again, make sure the numbers add up*
In this case we expect 430 rows to be created. And of course 


## Define the model
All models must be defined within the TamcGlobal namespace because the tamcglobal database has all types of data, whereas TamcWatch does not have alerts for example. To add your model to the TamcGlobal namespace, create `app/models/tamc_global/users.rb` with the following

```
module TamcGlobal
  class User < ApplicationRecord
    establish_connection TAMC_GLOBAL_DB
    self.table_name = 'user'
  end
end
```

To make the same model accessible for the TamcWatch database, create a the same file in `app/models/tamc_watch/` and replace `TAMC_GLOBAL_DB` with `TAMC_WATCH_DB`.

From here you can specify TamcGlobal::User or TamcWatch::User but you should use $db_namespace::User and the correct namespace will be selected. This is a global variable we set.

## Create a Sync class for the model

# Views Instead of Tables?
- User should be table because its accessed frequently
- Alerts should be a table because there are a lot
- But some tables (e.g. Dispatches) are not accessed frequently and only have a couple thousand rows. So we can just make this a database view for now and convert it to a table later?

https://dan.chak.org/enterprise-rails/chapter-11-view-backed-models/