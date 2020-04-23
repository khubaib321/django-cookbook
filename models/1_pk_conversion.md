# Changing primary key field type

## Problem
Django by default uses `AutoField` (integer, auto increment) as a primary key for a new model. Suppose we are migrating data from another database that uses different kind of primary keys in its tables such as a `UUID` or `GUID` field. To import such data into a Django database table and avoid any data loss, we created a model and provided our own primary key to match the data type. For example:  
`id = models.UUIDField(primary_key=True, default=uuid4, editable=False)`  

Now the problem is we need to convert this `UUID` field into a Django `AutoField` so that it is consistent with default Django database models and in doing so avoid any migration errors. If we simply change it to `models.UUIDField` and attempt to migrate, Django will throw `data integrity error` and will not apply the migration. This is because `UUID` is a string of random characters ([a-z][0-9]) and we are trying to convert that column into an integer type when direct conversion is not possible in this case.

## Solution #1
One way is to make a new ID field in your model, then convert it to primary key, and remove the `UUID` field.
1. Create an Integer field in your model (`new_id`) and make migration (do not migrate yet).  
`new_id = models.IntegerField(null=True)`  
2. Open migration file and add code to populate the `new_id` field with proper auto increment values.
Override `Migration.apply` method in your migration class. Let's say our model name is `Todo`:  
```
    def apply(self, project_state, schema_editor, collect_sql=False):
        """Override Migration.apply"""

        project_state = super(Migration, self).apply(project_state, schema_editor, collect_sql)

        # Populate new_id field with AutoField data (auto increment values)
        queryset = Todo.objects.all().order_by('date_created')
        for index, todo in enumerate(queryset):
            todo.new_id = index + 1
            todo.save()

        return project_state
```
3. Now apply migration and see the `new_id` field in your table with proper auto increment values.
4. Since we have proper auto incrment values in our `new_id` field, we can safely change it's type to `AutoField`. We can also now make it our new primary key as it has unique values that won't violate any primary key constraints.   
`id = models.UUIDField(primary_key=False, default=uuid4, editable=False)`  
`new_id = models.AutoField(primary_key=True, serialize=False, verbose_name='ID')`  
5. Now applying migration with above changes, Django might also ask for a default value but it can be ignored as our field is already populated with auto increment values. We now have `new_id` as our primary key and the `id` is now just a normal field.
6. Now we can remove the `id` property from our model and rename `new_id` to take its place.  
`id = models.AutoField(primary_key=True, serialize=False, verbose_name='ID')`  
7. Apply migration with above changes and see in database that we only have one `id` column as primary key and correct auto incremented values.
8. Add `auto_created=True` to the `id` property.  
`id = models.AutoField(auto_created=True, primary_key=True, serialize=False, verbose_name='ID')`  
9. Apply migration then remove the `id` field from model. Removing it will not have any effect now because Django by default creates `AutoField` `id` for primary key and we have replicated just that.

It looks like we could have done Step #6 and Step #8 and apply migration only once but if we combine these steps then instead of renaming `new_id` column to `id`, Django will drop the `new_id` column and create new one named `id`. This can be seen in the resulting migration file that Django creates. We do not want that because we have already populated our `new_id` column  and we just need a `migrations.RenameField` operation.

## Solution #2
Another way is to create a new Model with same table columns as properties but not explicitly providing the `id` field. Django will then create `AutoField` `id` by itself since that is the default behaviour. We can then copy all data from old table to new table using SQL query like  
```
INSERT INTO new_table (column1, column2, column3)
SELECT column1, column2, column3 FROM old_table
```
This method on paper looks simpler but it involves manually migrating data from one table to another. It will also require to duplicate your old model code and any associated logic as well which is more risky, error prone and may require extensive testing.
