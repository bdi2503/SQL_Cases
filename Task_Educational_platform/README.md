# Educational platform
### Purpose:
**1)** Calculate the number of students who have correctly solved 20 tasks for the current month.
```
query = """
 SELECT COUNT (st_id) as count_diligent_students
 FROM (SELECT st_id , SUM(correct) as summa
       FROM default.peas
       WHERE  correct  > 0
       GROUP BY st_id  
       HAVING summa >= 20)
"""
cds = ph.read_clickhouse(query, connection=connection_default)
print('Количество усердных студентов с 30 по 31 октября составило', cds.count_diligent_students[0], 'человек.')
```
The number of diligent students from October 30th to October 31st was 136.

**2)** To calculate the main metrics of the educational platform in one query.
Based on the data from the educational platform, I calculated the main metrics in one query:
- ARPU; 
- ARPU; 
- CR to purchase; 
- CR from the active user's to the purchase; 
- The user's CR from math activity to the purchase of a math course.

![image](https://github.com/user-attachments/assets/8ede9ce3-c984-45e4-9c59-0c2c4d42329a)


The complete solution is in the 
**[educational_platform_burykin_dmitriy.ipynb](https://github.com/bdi2503/SQL_Cases/blob/main/Task_Educational_platform/educational_platform_burykin_dmitriy.ipynb/ "link project")** 
file. The data is in the 
**[data](https://github.com/bdi2503/SQL_Cases/tree/main/Task_Educational_platform/data/ "link data")** 
folder.

**Tool:** SQL (Clickhouse), Python (Pandahouse)
