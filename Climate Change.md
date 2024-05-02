## Climate Change
To practice what you’ve learned about window functions, you are going to use climate data from each state in the United States. A labeled map of the United States can be found here.

These data will show the average annual temperature for each state – this is the average temperature of every day in all parts of the state for that year.

For this project, you will be working with one table:

- state_climate
![cc00](images/cc00.png)

### Understanding the Data
1.
Let’s see what our table contains by running the following command:

```msql
SELECT * 
FROM state_climate
LIMIT 10;
 ```
![cc01](images/cc01.png)
