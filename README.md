# Arrest Data Analysis (2010–2019)

## Overview

This repository contains a detailed SQL analysis of arrest data from 2010 to 2019, featuring both basic and advanced queries to explore arrest patterns, trends, and insights across time, geography, demographics, and charge types.

---

## Dataset Description

The dataset contains:

- `report_id`: Unique report identifier  
- `report_type`: Booking or RFC  
- `arrest_date`: Date of arrest  
- `arrest_time`: Time of arrest  
- `area_id`, `area_name`: Geographic location  
- `age`: Age of individual arrested  
- `sex_code`: Gender code ('M' or 'F')  
- `charge_group_description`: Broad charge category  
- `arrest_type_code`: Arrest type (e.g., felony complaint filed)  
- `charge_description`: Detailed charge description  
- `address`: Location of arrest  
- `booking_date`, `booking_time`, `booking_location`: Booking details  

---

## Analysis Questions and Solutions

### Basic Questions

1. How many total arrests were made between 2010 and 2019?
   ```sql
SELECT 
COUNT(*) AS total_arrests
FROM ArrestData
WHERE YEAR(arrest_date) BETWEEN 2010 AND 2019
```
