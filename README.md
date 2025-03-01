# Hive - Employee and Department Data Analysis

This repository contains Hive queries and scripts to analyze employee and department data using partitioned tables.

## Objectives
### Problem Statement
You are provided with two datasets: `employees.csv` and `departments.csv`. The tasks involve:

1. Loading `employees.csv` into a temporary Hive table and then transforming it into a partitioned table.
2. Loading `departments.csv` into a Hive table.
3. Performing various queries to analyze employee and department data.

## Setup and Execution

### 1. **Start the Hadoop Cluster**
To initiate the Hadoop cluster, execute:

```bash
docker compose up -d
```

### 2. **Load Data into Temporary Hive Tables**

```sql
CREATE TABLE employees_temp (
    emp_id INT,
    name STRING,
    age INT,
    job_role STRING,
    salary INT,
    project STRING,
    join_date STRING,
    department STRING
) ROW FORMAT DELIMITED 
FIELDS TERMINATED BY ',' 
STORED AS TEXTFILE;

LOAD DATA INPATH 'hdfs:///user/hive/employees.csv' INTO TABLE employees_temp;

CREATE TABLE departments_temp (
    dept_id INT,
    department_name STRING,
    location STRING
) ROW FORMAT DELIMITED 
FIELDS TERMINATED BY ',' 
STORED AS TEXTFILE;

LOAD DATA INPATH 'hdfs:///user/hive/departments.csv' INTO TABLE departments_temp;
```

### 3. **Create Partitioned Table and Insert Data**

```sql
SET hive.exec.dynamic.partition = true;
SET hive.exec.dynamic.partition.mode = non-strict;

CREATE TABLE employees_partitioned (
    emp_id INT,
    name STRING,
    age INT,
    job_role STRING,
    salary INT,
    project STRING,
    join_date STRING
) 
PARTITIONED BY (department STRING)
ROW FORMAT DELIMITED 
FIELDS TERMINATED BY ',' 
STORED AS PARQUET;

ALTER TABLE employees_partitioned 
ADD PARTITION (department='HR')
PARTITION (department='Engineering')
PARTITION (department='Marketing')
PARTITION (department='Finance')
PARTITION (department='Sales');

INSERT OVERWRITE TABLE employees_partitioned PARTITION (department)
SELECT emp_id, name, age, job_role, salary, project, join_date, department 
FROM employees_temp;

SELECT DISTINCT department FROM employees_partitioned;
```

### 4. **Querying the Data**

#### Retrieve employees who joined after 2015
```sql
INSERT OVERWRITE DIRECTORY '/user/hive/output/EmployeesJoinedAfter2015'
SELECT * FROM employees_partitioned WHERE year(join_date) > 2015;
```

#### Find the average salary per department
```sql
INSERT OVERWRITE DIRECTORY '/user/hive/output/Employeessalaryaverageeachdepartment'
SELECT department, AVG(salary) AS avg_salary FROM employees_partitioned GROUP BY department;
```

#### Identify employees working on the 'Alpha' project
```sql
INSERT OVERWRITE DIRECTORY '/user/hive/output/projectalpha'
SELECT * FROM employees_partitioned WHERE project = 'Alpha';
```

#### Count employees per job role
```sql
INSERT OVERWRITE DIRECTORY '/user/hive/output/Employeesjobrole'
SELECT job_role, COUNT(*) AS count FROM employees_partitioned GROUP BY job_role;
```

#### Retrieve employees earning above the average salary of their department
```sql
INSERT OVERWRITE DIRECTORY '/user/hive/output/greaterthanaverage'
SELECT * FROM employees_partitioned e 
WHERE salary > (SELECT AVG(salary) FROM employees_partitioned WHERE department = e.department);
```

#### Find the department with the highest number of employees
```sql
INSERT OVERWRITE DIRECTORY '/user/hive/output/highestemployees'
SELECT department, COUNT(*) AS employee_count 
FROM employees_partitioned GROUP BY department 
ORDER BY employee_count DESC LIMIT 1;
```

#### Exclude employees with null values
```sql
INSERT OVERWRITE DIRECTORY '/user/hive/output/Employeeswithnull'
SELECT * FROM employees_partitioned WHERE emp_id IS NOT NULL AND name IS NOT NULL AND age IS NOT NULL 
AND job_role IS NOT NULL AND salary IS NOT NULL AND project IS NOT NULL AND join_date IS NOT NULL AND department IS NOT NULL;
```

#### Join employees and departments to get department locations
```sql
INSERT OVERWRITE DIRECTORY '/user/hive/output/Employeeswithdepartmentlocations'
SELECT e.*, d.location FROM employees_partitioned e INNER JOIN departments_temp d ON e.department = d.department_name;
```

#### Rank employees within each department by salary
```sql
INSERT OVERWRITE DIRECTORY '/user/hive/output/Employeesineachdepartmentonsalary'
SELECT emp_id, name, department, salary, 
RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS salary_rank 
FROM employees_partitioned;
```

#### Find the top 3 highest-paid employees in each department
```sql
INSERT OVERWRITE DIRECTORY '/user/hive/output/highestpaid3eachdepartment'
SELECT * FROM (
    SELECT emp_id, name, department, salary, 
    RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS rank 
    FROM employees_partitioned
) ranked WHERE rank <= 3;
```

### 5. *Execute Queries and Save Output*
Run each command individually. The outputs of the commands are stored in the output folder with the given names.


### 6. *Access Hive Server Container*
bash
docker exec -it hive-server /bin/bash


### 7. *Copy Output from HDFS to Local Filesystem (Inside the Container)*
bash
hdfs dfs -get /user/hive/output /tmp/output


### 8. *Exit the Container*
bash
exit

### 9. **Exit the Container**
```bash
exit
```

### 10. **Check Current Working Directory on the Host**
```bash
pwd
```

### 11. **Copy Output Files from Docker Container to Host Machine**
```bash
docker cp hive-server:/tmp/output /........(Use the directory path obtained from `pwd`)
```

### 12. **Commit Changes to GitHub**
```bash
git add .
git commit -m "Updated analysis results"
git push origin main
```

### Challenges Faced
1. **Configuring Hive with Partitioning:** Setting up and managing partitioned tables required careful handling to ensure efficient query execution.
2. **Handling Large Datasets:** Processing employee and department data in Hive required optimizations to improve performance.
3. **Dynamic Partitioning Issues:** Enabling and correctly implementing dynamic partitioning posed challenges in data insertion.
4. **Join Performance Optimization:** Optimizing the join operations between `employees_partitioned` and `departments_temp` to improve query speed.
5. **Extracting Meaningful Insights:** Designing queries to extract relevant insights, such as salary comparisons and department-based rankings, required iterative refinements.

