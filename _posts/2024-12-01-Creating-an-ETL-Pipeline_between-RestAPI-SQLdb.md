---
layout: post
title: Extract data from RapidAPI, transform it, and load it into a MySQL database.
image: "/posts/ETL-Pipeline.jpg" 
tags: [Python, ETL, Pipeline, Rest API]
---
This project demonstrates how to:

> Extract real estate rental data from RapidAPI’s Rental Estimate API.
> Transform the data into a structured format for analysis.
> Load the processed data into a MySQL database.

By the end of this project, we will have a structured real estate rental database that can be used for market analysis, price predictions, or integration into an analytics dashboard.

Real estate is a data-driven industry. Whether you're an investor, agent, or renter, access to accurate pricing data can make or break decisions. The problem? Data is often scattered across multiple sources or behind paywalls.

With this ETL pipeline, I: ✅ Automate Data Collection (No more manual scraping!)
✅ Normalize & Structure the Data (Ready for analysis)
✅ Store in MySQL (For querying & insights)

This is not just another API project—this is data science applied to the real world.
---
Step 1: Extract Data from RapidAPI
The Data Source
For this project, I chose RapidAPI’s Rental Estimate API because it provides real-time rental price estimates and comparable property listings.

What We Are Extracting
We will request data for a 4-bedroom house in San Antonio, TX and retrieve:

Estimated rent price
Comparable properties (location, price, size, and more)

```python
import requests
import json

# API Credentials
API_URL = "https://realtymole-rental-estimate-v1.p.rapidapi.com/rentalPrice"
HEADERS = {
    "x-rapidapi-key": "YOUR_RAPIDAPI_KEY",  # Replace with your API key
    "x-rapidapi-host": "realtymole-rental-estimate-v1.p.rapidapi.com"
}

# API Query Parameters
PARAMS = {
    "address": "5500 Grand Lake Drive, San Antonio, TX, 78244",
    "propertyType": "Single Family",
    "bedrooms": 4,
    "bathrooms": 2,
    "squareFootage": 1600,
    "compCount": 5
}

# Function to Extract Data
def extract_data():
    response = requests.get(API_URL, headers=HEADERS, params=PARAMS)
    
    if response.status_code == 200:
        return response.json()  # Return JSON response
    else:
        print("Failed to fetch data:", response.status_code, response.text)
        return None
# Fetch Data
api_data = extract_data()
---

Explanation:
> We define the API URL and required headers.
> API parameters specify the property details.
> extract_data() sends a GET request and returns a JSON response.
```

Step 2: Transform Data
Now, we will extract key fields from the API response and prepare them for MySQL.

Why Transformation is Necessary?

> Raw API data is often nested and unstructured.
> Some fields are unnecessary for analysis.
> Dates need formatting for MySQL.
```

```python
from datetime import datetime

def transform_data(api_data):
    if not api_data:
        return None, None

    # Extract high-level rental details
    rental_info = {
        "rent": api_data["rent"],
        "rentRangeLow": api_data["rentRangeLow"],
        "rentRangeHigh": api_data["rentRangeHigh"],
        "longitude": api_data["longitude"],
        "latitude": api_data["latitude"]
    }

    # Extract individual property listings
    listings_data = []
    for listing in api_data.get("listings", []):
        listings_data.append({
            "id": listing["id"],
            "formattedAddress": listing["formattedAddress"],
            "longitude": listing["longitude"],
            "latitude": listing["latitude"],
            "city": listing["city"],
            "state": listing["state"],
            "zipcode": listing["zipcode"],
            "price": listing["price"],
            "publishedDate": listing["publishedDate"],
            "distance": listing["distance"],
            "daysOld": listing["daysOld"],
            "correlation": listing["correlation"],
            "address": listing["address"],
            "county": listing["county"],
            "bedrooms": listing["bedrooms"],
            "bathrooms": listing["bathrooms"],
            "propertyType": listing["propertyType"],
            "squareFootage": listing.get("squareFootage", None),
            "lotSize": listing.get("lotSize", None),
            "yearBuilt": listing.get("yearBuilt", None)
        })
    
    return rental_info, listings_data

# Transform Data
rental_info, listings_data = transform_data(api_data)

Explanation:
> The function extracts rental price summary and individual listings.
> It ensures missing data fields are handled using get().

```

Step 3: Load Data into MySQL
> Now, we create a MySQL database and tables.
> These can then be further analyzed or if required we can use the data to create meaningful reports and dashboards to support making data driven changes and provide insights.


```python
import mysql.connector
from mysql.connector import Error

def connect_to_mysql():
    try:
        connection = mysql.connector.connect(
            host="localhost",
            user="pkalathas",
            password="YOUR_MYSQL_PASSWORD"
        )
        if connection.is_connected():
            print("Connected to MySQL")
            return connection
    except Error as e:
        print(f"Error: {e}")
        return None

def setup_database():
    connection = connect_to_mysql()
    cursor = connection.cursor()

    # Create Database
    cursor.execute("CREATE DATABASE IF NOT EXISTS RealEstateDB;")
    cursor.execute("USE RealEstateDB;")

    # Create Tables
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS RentalSummary (
            id INT AUTO_INCREMENT PRIMARY KEY,
            rent FLOAT,
            rentRangeLow FLOAT,
            rentRangeHigh FLOAT,
            longitude FLOAT,
            latitude FLOAT
        );
    """)

    cursor.execute("""
        CREATE TABLE IF NOT EXISTS Listings (
            id VARCHAR(255) PRIMARY KEY,
            formattedAddress TEXT,
            longitude FLOAT,
            latitude FLOAT,
            city VARCHAR(100),
            state VARCHAR(50),
            zipcode VARCHAR(20),
            price FLOAT,
            publishedDate DATETIME,
            distance FLOAT,
            daysOld FLOAT,
            correlation FLOAT,
            address VARCHAR(255),
            county VARCHAR(100),
            bedrooms INT,
            bathrooms INT,
            propertyType VARCHAR(100),
            squareFootage INT,
            lotSize INT,
            yearBuilt INT
        );
    """)

    print("Database and tables created successfully.")
    connection.commit()
    cursor.close()
    connection.close()

setup_database()

Explanation:
> Connects to MySQL and creates the database.
> Defines tables for rental summary and listings.
```

Step 4: Insert Data into MySQL
> Now, we insert transformed data into MySQL database that we created in the previous step.
> This will then allow us to perform further analysis either using the SQL libraries through python or directly from MySQL


```python
def load_data(rental_info, listings_data):
    connection = connect_to_mysql()
    cursor = connection.cursor()
    cursor.execute("USE RealEstateDB;")
    
    # Insert Rental Summary
    cursor.execute("""
        INSERT INTO RentalSummary (rent, rentRangeLow, rentRangeHigh, longitude, latitude)
        VALUES (%s, %s, %s, %s, %s);
    """, (rental_info["rent"], rental_info["rentRangeLow"], rental_info["rentRangeHigh"],
          rental_info["longitude"], rental_info["latitude"]))
    
    # Insert Listings
    for listing in listings_data:
        try:
            published_date = datetime.strptime(listing["publishedDate"], "%Y-%m-%dT%H:%M:%S.%fZ").strftime("%Y-%m-%d %H:%M:%S")
        except ValueError:
            published_date = None

        cursor.execute("""
            INSERT INTO Listings (id, formattedAddress, longitude, latitude, city, state, zipcode, price, 
                publishedDate, distance, daysOld, correlation, address, county, bedrooms, bathrooms, propertyType,
                squareFootage, lotSize, yearBuilt)
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s);
        """, tuple(listing.values()))

    connection.commit()
    cursor.close()
    connection.close()

load_data(rental_info, listings_data)
```
