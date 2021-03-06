Django支持通过指定的月，日，周几来搜索datetime, 如

yourobj.objects.filter(datetime_file__month=your_month)

这个是1.6之后加入的，是向后不兼容的[1]。

这个功能在sqlite3里面是正常的，但是在mysql中是搜索不到的，有人向django提问了[2]。

在ticket中最后给评论给出说明，即这个功能在不同的数据库下不一样[3], mysql下需要向mysql导入timezone信息的。

所以原因就是当你在django里面开启USE_TZ=True的时候，数据库中的datetime field会先转成目前时区的时间
(`When USE_TZ is True, datetime fields are converted to the current time zone before filtering. This requires time zone definitions in the database.`[4])
并且，SQL语句中会出现EXTRACT(MONTH FROM CONVERT_TZ(`LocalData_garmentchangelog`.`create_time`, 'UTC', 'Asia/Shanghai'))总是null，所以filter回来的结果就是空了。

解决方式就是向mysql导入时区信息，命令为[5]:
mysql_tzinfo_to_sql /usr/share/zoneinfo | mysql -u root -p mysql

最后，0000-00-00 00:00:00的时间在mysql中会为None，在loaddata的时候需要注意。

全面的解释在[6]:


.. [1] https://docs.djangoproject.com/en/dev/releases/1.6/#time-zone-aware-day-month-and-week-day-lookups

.. [2] https://code.djangoproject.com/ticket/22528

.. [3] https://docs.djangoproject.com/en/1.7/ref/models/querysets/#datetimes
.. 
.. [4] https://docs.djangoproject.com/en/1.7/ref/models/querysets/#month
.. 
.. [5] http://dev.mysql.com/doc/refman/5.6/en/mysql-tzinfo-to-sql.html
.. 
.. [6] http://www.zhiwehu.com/blog/django-filter-on-datetime__month-with-mysql-should/
