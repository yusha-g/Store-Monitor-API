# Store Monitor API

## TABLE OF CONTENTS
<b> [1. Introduction](https://github.com/yusha-g/Store-Monitor-API#1-introduction)  </b> <br/>

<b> [2. Implementation](https://github.com/yusha-g/Store-Monitor-API#2-implementation)  </b> <br/>
&nbsp;&nbsp;&nbsp;[2.1. Global Variables](https://github.com/yusha-g/Store-Monitor-API#21-global-variables) <br/>
&nbsp;&nbsp;&nbsp;[2.2. Functions](https://github.com/yusha-g/Store-Monitor-API#22-functions) <br/>
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;[Jump to: Interpolation & Uptime, Downtime calculation](https://github.com/yusha-g/Store-Monitor-API#227-interpolate_status) <br/>
&nbsp;&nbsp;&nbsp;[2.3. API Endpoints](https://github.com/yusha-g/Store-Monitor-API#23-api-endpoints) <br/>

<b> [3. Outputs and Results](https://github.com/yusha-g/Store-Monitor-API#3-outputs-and-results)  </b> <br/>
&nbsp;&nbsp;&nbsp;[3.1. Results](https://github.com/yusha-g/Store-Monitor-API#31-results) <br/>
&nbsp;&nbsp;&nbsp;[3.2. Video Demonstration](https://github.com/yusha-g/Store-Monitor-API#32-output-demonstration) <br/>

<b> [4.Installation and Running Guide](https://github.com/yusha-g/Store-Monitor-API#4-installation-and-running-guide)  </b> <br/>
&nbsp;&nbsp;&nbsp;[4.1. Installing Dependencies](https://github.com/yusha-g/Store-Monitor-API#41-installing-dependencies) <br/>
&nbsp;&nbsp;&nbsp;[4.2. Running the Application](https://github.com/yusha-g/Store-Monitor-API#42-running-the-application) <br/>



# 1. Introduction

## 1.1. Problem Statement

Several stores are monitored to check if they are online or not. All stores are supposed to be online during their business hours, but due to some unknown reason a store may go inactive for a few hours. Store owners might want to get a report of how often this happened in the past. 

This API helps achieve this goal by generating a CSV containing the store’s uptimes and downtimes. 

## 1.2. Data Sources

1. Every store is polled roughly every hour and the data about whether the store was active or not in a CSV.  
The CSV has 3 columns (`store_id, timestamp_utc, status`) where status is active or inactive.  All timestamps are in **UTC**
2. The business hours of all the stores - schema of this data is `store_id, dayOfWeek(0=Monday, 6=Sunday), start_time_local, end_time_local`
    1. These times are in the **local time zone**
    2. If data is missing for a store, assume it is open 24*7
3. Timezone for the stores - schema is `store_id, timezone_str`
    1. If data is missing for a store, assume it is America/Chicago
    2. This is used so that data sources 1 and 2 can be compared against each other. 

# 2. Implementation

## 2.1. Global Variables

### 2.1.1. report_id_lst

- List of all report IDs generated.
- Helps make sure no duplicate report IDs are generated.

### 2.1.2. generating_csv

- Flag to keep track of report generation.
- If False report is still generating. If True, report has finished generating.
- By default it is set to True.

## 2.2. Functions

### 2.2.1. get_connection

- Defined to establish connection with the MySQL database named ‘Work_DB’.
- The 3 data sources, 'Menu hours.csv', 'bq-results.csv', and 'store status.csv’ are stored in tables LK_menu_hours, LK_bq_result and LK_store_status respectively.

### 2.2.2. check_new_records

- This function is called from the route(’csv_update’)
- It reads the CSV files, compares the records with the existing records in the database tables, and inserts the new records if they don’t exist.
- It checks for new records every 1 hour.

### 2.2.3. get_stores

- Retrieves distinct store IDs from the 'LK_store_status' table within a specified time range.
- Since, we are to calculate uptime down downtime for the previous week, day and hour,
the time range is the current time and one week previous, since that is the maximum range we will be working with.

### 2.2.4. get_business_hours

- Retrieves the business hours (day, start time, end time) for a given store ID from the 'LK_menu_hours' table.
- If business hour is not provided for any day of the week, it is set to ('00:00:00', '23:59:59') i.e. 24*7
- The business hours are of the form { ”day_of_the_week”: ( ”start_time” , ”end_time” ) }

### 2.2.5. get_timezone

- Retrieves the timezone for a given store ID from the 'LK_bq_result' table.
- If no timezone is provided it is assumed to be 'America/Chicago’

### 2.2.6. **get_timestamp_in_business_hours**

- Retrieves timestamp and corresponding status for a given store ID from the 'LK_store_status' table and orders it by timestamp.
- Converts the UTC timestamp to the local timezone
- Checks if it falls within the specified business hours.
- The returned data is a list of tuples and is of the form [ (Timestamp, status) ]

### 2.2.7. **interpolate_status**

- interpolates the status for different time intervals (1 week, 1 day, 1 hour) based on the available status ranges.
- status_range contains the list of timestamps and statuses retrieved from the function get_timestamp_in_business_hours()
- Considering the 1 hour interval, the function works as follows:
    1. The time interval ranges from start_time to end_time
    2. Set the value of end_time to current time (or maximum timestamp value).
    3. Set the value of current_time to end_time minus 1 hour [1 day / 1 week].
    4. Create an empty list called interpolated_time to store interpolated time values.
    5. Repeat while current_time is less than end_time
        1. Iterate over each timestamp in status_range except the last timestamp. 
            1. Check if current_time falls between two consecutive timestamps — i.e. timestamp ‘i’ (inclusive) and timestamp ‘i+1’ (exclusive) of status_range
                1. If it does, append current time it to interpolated_time and set it’s status to lower timestamp’s ( ’i’ )status
                2. Else, append the upper timestamp (’i+1’) to interpolate_time.
                This is why we don’t iterate over the last element, since, it will produce duplicates. 
            2. Increment current_time by 1 minute [1 hours / 1 day]
    6. Return interpolated_time  
- All in all, we return three lists:
    - interpolated_time_1h: containing 60 points
    - interpolated_time_1d: containing 24 points
    - interpolated_time_1w: containing 7 points

### 2.2.8. get_uptime_downtime

- Calculates the uptime and downtime based on the interpolated time and status.
- Iterate through interpolated time and status obtained from function interpolate_status()
- If status is ‘active’ increment uptime, else increment ‘downtime’ and return them

### 2.2.9. generate_report_id

- generates unique report id

### 2.2.10. generate_report

- Called from route(’/trigger_report’). It generates a report by calling various functions and performing database queries.
- Retrieves maximum timestamp in LK_store_status and Get all unique store_ids by using `get_stores()`
- Generates report id by calling `generate_report_id()`
- Checks if report id was previously generated using report_id_lst.
If it is was previously generated, regenerate it, else continue.
- Create csv file with report id having the following schema
`store_id, uptime_last_hour(in minutes), uptime_last_day(in hours), update_last_week(in hours), downtime_last_hour(in minutes), downtime_last_day(in hours), downtime_last_week(in hours)`
- For each unique store_id
    - get business hours using `get_business_hours()`
    - get time zone using `get_timezone()`
    - Retrieve observed timestamps and corresponding status within business hours from LK_store_status using `get_timestamp_in_business_hours()`
    - get interpolated times by 1 week, 1 day and 1 hour using `interpolate_status()`
    - get uptime and downtime for 1 week by calling `get_uptime_downtime()` with interpolated_time_status_1w.
    - Similarly, get uptime and downtime for 1 day and 1 hour.
    - Write into CSV.
- Once CSV is generated, set csv generation flag to True, indicating that the report has completed generating.

## 2.3. API Endpoints

### 2.3.1. (’/’)

- Renders the landing page.
- Shows a list of all the generated report IDs

### 2.3.2. (’/trigger_report’)

- Calls the `generate_report()` function
- Returns the report_id

### 2.3.3. (’/get_report/<report_id>’)

- Based on the inputted report_id checks:
    - If CSV is still generating: Output ‘Running’
    - If CSV has finished generating: Output ‘Completed’.
        - Render a webpage showing a preview of the generated CSV.
        - Provide a link to download the entire CSV file

### 2.3.4. (’/csv_update’)

- Calls the `check_new_records()` function

# 3. Outputs and Results

## 3.1. Results

- In the landing page, the user is able to view all the report_ids
- On triggering report generation using ‘/trigger_report’ new report_id is generated and displayed on the webpage.
- We can check the status of report generation with report_id using ‘/generate_report/<report_id>’.
- If report is still generating, outputs ‘Running…’
- Else outputs:
    - ‘Completed’
    - A download link to download the generated report_id
    - A preview of the generated csv with it’s top 20 records.
- While newer reports are generating, previously generated reports can also be accessed using their report IDs.

## 3.2. Output Demonstration


https://github.com/yusha-g/Store-Monitor-API/assets/110189579/b8a9f837-2e7e-4180-978c-ce832620dd24


# 4. Installation and Running Guide

## 4.1. Installing Dependencies

1. Flask:
    - Open a terminal or command prompt.
    - Run the command **`pip install flask`** to install Flask.
2. Pandas:
    - Open a terminal or command prompt.
    - Run the command **`pip install pandas`** to install Pandas.
3. Pytz:
    - Open a terminal or command prompt.
    - Run the command **`pip install pytz`** to install Pytz.
4. Mysql-connector:
    - Open a terminal or command prompt.
    - Run the command **`pip install mysql-connector-python`** to install the MySQL connector for Python.
5. Datetime, Csv, os, Time:
    - These modules are part of the Python standard library, so no additional installation is required.

## 4.2. Running the Application

In the terminal, navigate to the folder in which the index.py is using the ‘cd’ command. Once you are in the directory, run the application by typing the following command: 

`python store_monitor.py`

This will start the Flask development server and the application will be available at [http://localhost:5000](http://localhost:5000/)

You can stop the program by pressing CTRL+C
