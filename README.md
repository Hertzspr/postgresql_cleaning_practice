# postgresql_cleaning_practice
Data cleaning on Nashville Housing in PostgreSQL

This is basically just the SQL code I wrote following Alex Freberg's project tutorial. 
But he did it in SQL Server while my code is in PostgreSQL. 
Data Source: https://github.com/AlexTheAnalyst/PortfolioProjects/blob/main/Nashville%20Housing%20Data%20for%20Data%20Cleaning.xlsx

What's it like doing it in PostgreSQL? 
+ Easier on Date type initial insertion to the table
- Different treatment of CTE, I couldn't use CTE as a direct reference to delete data, I needed to sort of join it first then pull an identifier to delete the intended rows.


