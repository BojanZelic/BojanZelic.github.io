---
layout: post
title:  "Kohana - Caching Database Columns"
date:   2014-11-30 18:11:26 -0700
categories: kohana php
---
Did you know that an extra query gets run every time you initialize a new model in Kohana?

It looks like this

```
SHOW COLUMNS FROM table_name
```

Personally I believe one of the more annoying "features" of Kohana's ORM is that caching doesn't occur with database models. When you enable debugging, & view some of the queries that are run, the "List Columns" query might actually be one of the heaviest.

If you're using multiple models on a page or if your database consists on a separate networked computer. It would make sense to reduce the extra trips to the database, & cache the results locally. Especially considering how easy it is to include this code in your project.

I found some code on their forums and wanted to share it. Considering they're moving their forums. This might be difficult to find in the future so I'm hoping this post may help somebody someday.
ORM.php

```
class ORM extends Kohana_ORM
{
    public function reload_columns($force = false)
    {
        if ($force === true || empty($this->_table_columns))
        {
            $table_column_cache_name = APPPATH . "cache/model_table_columns_" . $this->_object_name . ".php";
            // Check if we have a cache file
            if(file_exists($table_column_cache_name))
            {
                //Grab our table columns from the include
                $this->_table_columns = include  $table_column_cache_name;
            }
            else
            {
                //Grab column information from database
                $this->_table_columns = $this->list_columns(true);

                //Export our table columns as php source to the cache file
                file_put_contents($table_column_cache_name, "<?php return ". var_export($this->_table_columns, true) . ";");
            }
        }

        return $this;
    }
}
```

Basically all you need to do is extend the Kohana_ORM class with your own version & override the reload_columns function. Hopefully you automated migrations and all you'd need to do is insert the following line after they are run.
start.sh

```
#!/bin/bash
./minion migrations:run
rm ./application/cache/model_table_columns*
```

I've recently used this code in a project & found alot of success with it.

Credits: Kohana Forums