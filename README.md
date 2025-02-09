# Payroll_Anakysis_Dashboard

## Objective
This project implements a real-time data pipeline to process payroll data using Apache Flink and Flink SQL. The pipeline achieves the following goals:

- Process Avro-formatted payroll data from Kafka topics.
- Enrich and analyze the data in real-time using Flink.
- Store the processed data in Elasticsearch for visualization.
- Provide insights through a comprehensive Kibana dashboard

---

## Dataset Description
Three datasets represent key aspects of the payroll system and were generated from Avro schemas:

1. **Payroll Transactions (payroll1)**
- Description: Contains employee bonus details.
- Schema:
  - employee_id: Unique identifier for the employee.
  - bonus: Bonus amount for the employee.
  - ts: Event timestamp in epoch milliseconds.
  
Example Data:
   ```json
   {"employee_id":1006,"bonus":29,"ts":1622470994859}
   {"employee_id":1032,"bonus":44,"ts":1617672159259}
   ```

2. **Employee Details (payroll2)**
- Description: Contains personal and job-related details of employees.
- Schema:
  - employee_id: Unique identifier for the employee.
  - first_name: Employee's first name.
  - last_name: Employee's last name.
  - age: Age of the employee.
  - ssn: Social Security Number.
  - hourly_rate: Hourly wage.
  - gender: Gender of the employee.
  - email: Employee's email address.

Example Data:
   ```json
{"employee_id":1092,"first_name":"Chelsae","last_name":"Ballin","age":38,"ssn":"929-64-1842","hourly_rate":21,"gender":"female","email":"hevl@mycompany.com"}
{"employee_id":1003,"first_name":"Zeb","last_name":"Murrish","age":46,"ssn":"557-94-0484","hourly_rate":11,"gender":"male","email":"yfxw@mycompany.com"}
   ```

3. **Departmental and Arrival Details (payroll3)**
- Description: Contains lab assignments and department information for employees.
- Schema:
  - employee_id: Unique identifier for the employee.
  - lab: Lab assignment of the employee.
  = department_id: Department ID.
  = arrival_date: Date of arrival (days since 1970-01-01).

Example Data:
  ```json
{"employee_id":1098,"lab":"lab-8","department_id":9,"arrival_date":18031}
{"employee_id":1023,"lab":"lab-4","department_id":7,"arrival_date":18531}
   ```

--- 

## **Data Pipeline Process**

---

### **1. Data Generation**  
Data was generated from Confluent Avro schemas using the following commands:  
```bash
./gendata.sh payroll_bonus.avro xyz1.json 10000
./gendata.sh payroll_employee.avro xyz2.json 10000
./gendata.sh payroll_employee_location.avro xyz3.json 10000
```
- **`gendata.sh`**: Extracts data from the Avro schema and converts it to JSON.  
- **Input**: Avro schema.  
- **Output**: JSON file with random synthetic data.

---

### **2. Data Transformation**  
The JSON files were transformed into key-value pairs using the **`convert.py`** script to prepare them for Kafka ingestion:  
```bash
python $HOME/Documents/fake/convert.py
```

---

### **3. Kafka Ingestion**  
Data from the JSON files was streamed into Kafka topics using **`gen_sample.sh`**:  
```bash
./gen_sample.sh /home/ashok/Documents/gendata/rev_xyz1.json |  kafkacat -b localhost:9092 -t payroll1 -K:  -P
./gen_sample.sh /home/ashok/Documents/gendata/rev_xyz2.json |  kafkacat -b localhost:9092 -t payroll2 -K:  -P
./gen_sample.sh /home/ashok/Documents/gendata/rev_xyz3.json |  kafkacat -b localhost:9092 -t payroll3 -K:  -P
```
- **`gen_sample.sh`**: Streams transformed JSON data to Kafka topics.  
- **Kafka Topics Created**: `payroll1`, `payroll2`, `payroll3`.

---

### **4. Real-Time Analysis with Apache Flink**  
Apache Flink and Flink SQL were used for real-time streaming analysis.  

#### **Table Creation in Flink SQL:**

1. **Payroll Transactions Table:**

  ```sql
CREATE TABLE payroll1 (
    employee_id BIGINT, -- Unique identifier for the employee
    bonus INT, -- Bonus amount for the employee
    ts BIGINT, -- Original epoch timestamp
    readable_ts AS TO_TIMESTAMP(FROM_UNIXTIME(ts / 1000)) -- Computed column for standard timestamp
) WITH (
    'connector' = 'kafka', -- Kafka as the source connector
    'topic' = 'payroll1', -- Kafka topic name
    'scan.startup.mode' = 'earliest-offset', -- Start reading from the earliest offset
    'properties.bootstrap.servers' = 'kafka:9094', -- Kafka broker address
    'format' = 'json', -- Data format is JSON
    'json.timestamp-format.standard' = 'ISO-8601' -- For correct timestamp parsing
);
  ```

2. **Employee Details Table:**

 ```sql
CREATE TABLE payroll2 (
    employee_id BIGINT, -- Unique identifier for the employee
    first_name STRING, -- First name of the employee
    last_name STRING, -- Last name of the employee
    age INT, -- Age of the employee
    ssn STRING, -- Social Security Number
    hourly_rate INT, -- Hourly rate of the employee
    gender STRING, -- Gender of the employee
    email STRING -- Email address of the employee
) WITH (
    'connector' = 'kafka', -- Kafka as the source connector
    'topic' = 'payroll2', -- Kafka topic name
    'scan.startup.mode' = 'earliest-offset', -- Start reading from the earliest offset
    'properties.bootstrap.servers' = 'kafka:9094', -- Kafka broker address
    'format' = 'json', -- Data format is JSON
    'json.timestamp-format.standard' = 'ISO-8601' -- For correct timestamp parsing
);
 ```

3. **Departmental Details Table:**

  ```sql
CREATE TABLE payroll3 (
    employee_id BIGINT, -- Unique identifier for the employee
    lab STRING, -- Lab assignment for the employee
    department_id INT, -- Department ID
    arrival_date INT, -- Numeric date (days since 1970-01-01)
    arrival_ts AS TO_TIMESTAMP(FROM_UNIXTIME(arrival_date * 86400)) -- Computed column for arrival date as a timestamp
) WITH (
    'connector' = 'kafka', -- Kafka as the source connector
    'topic' = 'payroll3', -- Kafka topic name
    'scan.startup.mode' = 'earliest-offset', -- Start reading from the earliest offset
    'properties.bootstrap.servers' = 'kafka:9094', -- Kafka broker address
    'format' = 'json', -- Data format is JSON
    'json.timestamp-format.standard' = 'ISO-8601' -- For correct timestamp parsing
);
 ```

4. **Enriched Payroll View: Combines data from all three sources:**

  ```sql

CREATE VIEW all_payroll_data AS
SELECT 
    p1.employee_id,  -- employee_id from payroll1
    p1.bonus,  -- bonus from payroll1
    TO_TIMESTAMP(FROM_UNIXTIME(p1.ts / 1000)) AS bonus_ts,  -- Convert ts to timestamp for payroll1
    p2.first_name,  -- first_name from payroll2
    p2.last_name,   -- last_name from payroll2
    p2.age,         -- age from payroll2
    p2.ssn,         -- ssn from payroll2
    p2.hourly_rate, -- hourly_rate from payroll2
    p2.gender,      -- gender from payroll2
    p2.email,       -- email from payroll2
    p3.lab,         -- lab from payroll3
    p3.department_id, -- department_id from payroll3
    p3.arrival_date,  -- arrival_date from payroll3
    TO_TIMESTAMP(FROM_UNIXTIME(p3.arrival_date * 86400)) AS arrival_ts  -- Convert arrival_date to timestamp for payroll3
FROM payroll1 AS p1
LEFT JOIN payroll2 AS p2
    ON p1.employee_id = p2.employee_id
LEFT JOIN payroll3 AS p3
    ON p1.employee_id = p3.employee_id;
 ```

---

### **5. Data Insertion into Elasticsearch**  
Before creating the Kibana dashboard, the enriched data was inserted into an Elasticsearch index.

#### **Data Insertion into Elasticsearch:**
1. **Elasticsearch Table Creation**
 ```sql
   CREATE TABLE payroll_data_dashboard (
    employee_id BIGINT,
    bonus INT,
    bonus_ts TIMESTAMP(6),
    first_name STRING,
    last_name STRING,
    age INT,
    ssn STRING,
    hourly_rate INT,
    gender STRING,
    email STRING,
    lab STRING,
    department_id INT,
    arrival_date INT,
    arrival_ts TIMESTAMP(6)
) WITH (
    'connector' = 'elasticsearch-7',  -- Elasticsearch connector
    'hosts' = 'http://elasticsearch:9200',  -- Elasticsearch host
    'index' = 'payroll_data_dashboard',  -- Elasticsearch index name
    'document-id.key-delimiter' = '-',  -- Delimiter for document IDs
    'format' = 'json',  -- Data format
    'sink.bulk-flush.max-actions' = '1'  -- Immediate flush for testing
);
 ```


2. **Inserting Data into Elasticsearch Index**
 ```sql
INSERT INTO payroll_data_dashboard
SELECT 
    employee_id,
    bonus,
    bonus_ts,
    first_name,
    last_name,
    age,
    ssn,
    hourly_rate,
    gender,
    email,
    lab,
    department_id,
    arrival_date,
    arrival_ts
FROM all_payroll_data;
 ```
---

### **6. Dashboard Creation with Kibana**  
The data in the **`payroll_data_dashboard`** index was visualized using Kibana.  
- An index pattern for **`player_game_dashboard`** was created in Kibana.  
- A detailed dashboard was designed to display payroll bonus, age, hourly_rate etc.

# Dashboard Analysis

![Docker_Flink](https://github.com/RishiGoswami836/Payroll_Analysis_Dashboard/blob/main/Dashboard.jpg)

---

### Video

https://github.com/user-attachments/assets/39a37e30-b45d-44af-bf23-bbf5131a6ca2

---

#### **1. Bonus Values by Arrival Time (Bar Chart)**
- **Observation:** The average bonus varies across different arrival times, with noticeable peaks on specific dates.
- **Insights:**
  - Employees arriving at certain times tend to receive higher bonuses, potentially indicating a correlation between arrival time and performance or productivity.
  - This trend can be used to adjust shift timings or optimize workforce allocation.

---

#### **2. Bonus Metrics (Summary Card)**

- **Observation:**
50th percentile bonus: 25
Average bonus: 25.813
Maximum bonus: 49
- **Insights:**
  - The distribution shows moderate variability, with most employees receiving bonuses close to the average.
  - This indicates equitable bonus distribution, but outliers may warrant further investigation.

---

#### **3. Average Hourly Rate per Gender (Area Chart)**

- **Observation:** Females have a higher average hourly rate compared to males.
- **Insights:**
  - This highlights a positive effort toward reducing gender wage gaps.
  - Additional analysis could verify if job roles and responsibilities are evenly distributed across genders.

 ---

 #### **4. Average Hourly Rate (Gauge)**
 
- **Observation:** The average hourly rate is 15.997.
- **Insights:**
  - This is a key metric for evaluating overall labor costs.
  - A slight increase in the average rate may enhance employee retention and job satisfaction.
 
----

#### **5. Hourly Rate by Age (Bar Chart)**

- **Observation:** Younger employees tend to have a lower hourly rate, while mid-aged employees dominate the higher hourly rate brackets.
- **Insights:**
  - This indicates experience plays a significant role in compensation.
  - Training younger employees may bridge skill gaps and lead to equitable pay distribution over time.
 
---

#### **6. Department-wise Bonus (Line Chart)**

- **Observation:** Bonuses fluctuate significantly across departments, with some departments receiving consistently higher bonuses.
- **Insights:**
  - Performance-based bonus allocation appears to be in practice.
  - Underperforming departments could benefit from targeted support and resource optimization.

---

#### **7. Bonus by Top 10 First Names (Area Chart)**

- **Observation:** Certain individuals (e.g., with names like "Adele" or "Byron") receive higher bonuses.
- **Insights:**
  - This may highlight high performers, suggesting they hold critical roles in the organization.
  - Consider sharing their best practices with others to elevate overall performance.
 
---

#### **8. Age-wise Gender Count (Bar Chart)**

- **Observation:** Gender representation is balanced across age groups, with slight variations.
- **Insights:**
  - This balance reflects a diverse workforce.
  - Continuous monitoring can ensure inclusivity across age brackets and job roles.

---

#### **9. Average Age by Arrival Time (Line Chart)**

- **Observation:** The average age of employees remains consistent over time, with slight fluctuations.
- **Insights:**
  - This suggests stable hiring practices with no sudden shifts in workforce demographics.
  - Consistent age distribution ensures sustained knowledge transfer between younger and older employees.

---

#### **10. Average Age and Hourly Rate (Gauge)**

- **Observation:**
Average age: 40.169
Average hourly rate: 15.997
- **Insights:**
  - Both metrics align with standard workforce benchmarks for industries with similar demographics.
  - Further segmentation (e.g., by role or department) could uncover actionable insights.

---

### **Managerial Insights**

1. **Employee Engagement:**
    - Understand bonus distribution patterns and identify top-performing employees.
    - Enhance engagement through targeted incentives.

2. **Departmental Analysis:**
    - Analyze lab and department performance based on arrival and bonus trends.

3. **Resource Allocation:**
    - Optimize workforce planning and resource allocation based on real-time insights.

---

