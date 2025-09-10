# zillow-data-engineering-pipeline

```
windows
cd Documents

mkdir zillow

# Step 1: Create the environment
python -m venv etl

# Step 2: Activate the environment
etl\Scripts\activate

code .
```

from the documentation in zillow api  we should have this as the docs

```
import requests

url = "https://zillow56.p.rapidapi.com/search"

querystring = {"location":"houston, tx","output":"json","status":"forSale","sortSelection":"priorityscore","listing_type":"by_agent","doz":"any"}

headers = {
	"x-rapidapi-key": "1ad87503b5msh3f1c9c47ceccc73p11a365jsn7f580a8061b8",
	"x-rapidapi-host": "zillow56.p.rapidapi.com"
}

response = requests.get(url, headers=headers, params=querystring)

print(response.json())
```

🔹 If something went wrong (like a wrong API key or bad internet), we print an error with the code (like 401, 404, etc.)



| Code | Meaning             | What it Means for Us                     |
| ---- | ------------------- | ---------------------------------------- |
| 200  | ✅ OK                | Everything worked fine. We got the data. |
| 401  | ❌ Unauthorized      | API key is missing or incorrect.         |
| 404  | ❌ Not Found         | The city name is wrong or not found.     |
| 500  | ❌ Server Error      | Something went wrong on their side.      |
| 429  | ❌ Too Many Requests | You called the API too many times.       |



Using the above scripts we will  have our fisrt etl pipeline and we will be getting our data   and then transforming it and loading it to a csv file we will have to create a file name called config.json for security of our api keys

```
import http.client
import json
import csv
import os
from datetime import datetime
from urllib.parse import quote

# List of cities to loop through
cities = [
    "houston, tx",
    "dallas, tx",
    "austin, tx",
    "san antonio, tx",
    "chicago, il",
    "los angeles, ca",
    "new york, ny",
    "miami, fl",
    "phoenix, az",
    "seattle, wa"
]

def extract_zillow_data(city):
    # Load API config
    with open('config.json', 'r') as f:
        config = json.load(f)
    
    headers = {
        'x-rapidapi-key': config['x-rapidapi-key'],
        'x-rapidapi-host': config['x-rapidapi-host']
    }
    
    # API parameters
    params = {
        "location": city,
        "output": "json",
        "status": "forSale",
        "sortSelection": "priorityscore",
        "listing_type": "by_agent",
        "doz": "any"
    }
    
    # Build query
    query = "&".join([f"{k}={quote(str(v))}" for k,v in params.items()])
    url = f"/search?{query}"
    
    # Make request
    conn = http.client.HTTPSConnection("zillow56.p.rapidapi.com")
    conn.request("GET", url, headers=headers)
    
    res = conn.getresponse()
    data = json.loads(res.read().decode("utf-8"))
    return data

# Create single CSV file
timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
filename = f"zillow_all_cities_{timestamp}.csv"
filepath = os.path.join(os.getcwd(), filename)

# Get all fieldnames from all cities first
all_fieldnames = set()
all_properties = []

for city in cities:
    print(f"Fetching data for {city}...")
    
    data = extract_zillow_data(city)
    properties = data.get('results', data.get('props', [data] if isinstance(data, dict) else data))
    
    if properties:
        # Add city name to each property
        for prop in properties:
            prop['city'] = city
            all_properties.append(prop)
            all_fieldnames.update(prop.keys())

# Write to single CSV
with open(filepath, 'w', newline='', encoding='utf-8') as f:
    writer = csv.DictWriter(f, fieldnames=sorted(all_fieldnames))
    writer.writeheader()
    
    for prop in all_properties:
        row = {field: prop.get(field, '') for field in all_fieldnames}
        writer.writerow(row)

print(f"All data saved to: {filepath}")
print(f"Total records: {len(all_properties)}")
print(f"Total columns: {len(all_fieldnames)}")
```

we will have our docker in our docker-compose.yml
```
version: '3'
services:
  postgres:
    image: postgres:14
    container_name: postgres_db
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: postgres
    ports:
      - "5432:5432"
    volumes:
      - ./dags:/usr/local/airflow/dags
      - ./dags/credential.txt:/usr/local/airflow/dags/config.json
volumes:
  postgres_data:
```
we willl have to initilize out fist dag ..extract tansfrom and load to postgres 
```
# Import core Airflow components
from airflow import DAG
from airflow.providers.postgres.hooks.postgres import PostgresHook
from airflow.decorators import task

# Import standard libraries
from datetime import datetime, timedelta
import http.client
import json
from urllib.parse import quote

# List of cities to collect Zillow data for
CITIES = [
    "houston, tx",
    "dallas, tx",
    "austin, tx",
    "san antonio, tx",
    "chicago, il",
    "los angeles, ca",
    "new york, ny",
    "miami, fl",
    "phoenix, az",
    "seattle, wa"
]

# Airflow connection IDs
POSTGRES_CONN_ID = "postgres_default"

# Default parameters for the DAG
default_args = {
    "owner": "airflow",
    "start_date": datetime(2025, 6, 21),
    "retries": 3,
    "retry_delay": timedelta(minutes=5),
}

# Function to load API key from config
def load_api_key():
    with open("/usr/local/airflow/dags/config.json", "r") as file:
        return json.load(file)

# Helper function to safely convert to numeric values
def safe_numeric(value, default=0, numeric_type=float):
    """Safely convert value to numeric type, handling empty strings and None"""
    if value is None or value == "":
        return default
    try:
        return numeric_type(value)
    except (ValueError, TypeError):
        return default

# Define the DAG
with DAG(
    dag_id="zillow_etl_pipeline",
    default_args=default_args,
    schedule="@daily",  # <-- fixed Airflow param (schedule_interval instead of schedule)
    catchup=False,
    tags=["zillow", "postgres", "etl"]
) as dag:

    # Task 1: Extract Zillow data
    @task()
    def extract_zillow_data():
        api_config = load_api_key()
        headers = {
            "x-rapidapi-key": api_config["x-rapidapi-key"],
            "x-rapidapi-host": api_config["x-rapidapi-host"],
        }

        all_city_data = []

        for city in CITIES:
            try:
                print(f"Fetching data for {city}...")

                params = {
                    "location": city,
                    "output": "json",
                    "status": "forSale",
                    "sortSelection": "priorityscore",
                    "listing_type": "by_agent",
                    "doz": "any",
                }

                query = "&".join([f"{k}={quote(str(v))}" for k, v in params.items()])
                url = f"/propertyExtendedSearch?{query}"

                conn = http.client.HTTPSConnection("zillow-com1.p.rapidapi.com")
                conn.request("GET", url, headers=headers)

                res = conn.getresponse()
                data = json.loads(res.read().decode("utf-8"))

                properties = data.get(
                    "results", data.get("props", [data] if isinstance(data, dict) else data)
                )

                if properties:
                    for prop in properties:
                        prop["city"] = city
                        all_city_data.append(prop)

                print(f"Extracted {len(properties)} properties from {city}")

            except Exception as e:
                print(f"Error extracting data for {city}: {str(e)}")
                continue

        print(f"Total properties extracted: {len(all_city_data)}")
        return all_city_data

    # Task 2: Transform the data
    @task()
    def transform_zillow_data(properties_list):
        transformed_data = []

        if not properties_list:
            print("No properties to transform!")
            return transformed_data

        print(f"Starting transformation of {len(properties_list)} properties")

        for i, prop in enumerate(properties_list):
            if "message" in prop:  # skip API error responses
                continue

            zpid_value = prop.get("zpid")
            if not zpid_value:
                continue

            try:
                transformed_prop = {
                    "zpid": safe_numeric(zpid_value, 0, int),
                    "streetAddress": prop.get("streetAddress", ""),
                    "city": prop.get("city", ""),
                    "state": prop.get("state", ""),
                    "zipcode": prop.get("zipcode", ""),
                    "country": prop.get("country", "USA"),
                    "bedrooms": safe_numeric(prop.get("bedrooms"), 0),
                    "bathrooms": safe_numeric(prop.get("bathrooms"), 0),
                    "homeStatus": prop.get("homeStatus", ""),
                    "price": safe_numeric(prop.get("price"), 0),
                    "taxAssessedValue": safe_numeric(prop.get("taxAssessedValue"), 0),
                    "zestimate": safe_numeric(prop.get("zestimate"), 0),
                    "rentZestimate": safe_numeric(prop.get("rentZestimate"), 0),
                    "daysOnZillow": safe_numeric(prop.get("daysOnZillow"), 0, int),
                    "latitude": safe_numeric(prop.get("latitude"), 0),
                    "longitude": safe_numeric(prop.get("longitude"), 0),
                    "propertyType": prop.get("propertyType", ""),
                    "livingArea": safe_numeric(prop.get("livingArea"), 0),
                    "lotAreaValue": safe_numeric(prop.get("lotAreaValue"), 0),
                    "yearBuilt": safe_numeric(prop.get("yearBuilt"), 0, int),
                    "timeOnZillow": safe_numeric(prop.get("timeOnZillow"), 0, int),
                    "currency": prop.get("currency", "USD"),
                    "datePriceChanged": safe_numeric(prop.get("datePriceChanged"), 0, int),
                    "extracted_date": datetime.now(),
                }
                transformed_data.append(transformed_prop)
            except Exception as e:
                print(f"Error transforming property {i}: {e}")
                continue

        print(f"Successfully transformed {len(transformed_data)} properties")
        return transformed_data

    # Task 3: Load data into PostgreSQL with UPSERT
    @task()
    def load_zillow_data(transformed_data):
        if not transformed_data:
            print("No data to load!")
            return

        pg_hook = PostgresHook(postgres_conn_id=POSTGRES_CONN_ID)
        conn = pg_hook.get_conn()
        cursor = conn.cursor()

        cursor.execute("""
        CREATE TABLE IF NOT EXISTS zillow_properties (
            zpid BIGINT PRIMARY KEY,
            streetAddress VARCHAR(500),
            city VARCHAR(100),
            state VARCHAR(50),
            zipcode VARCHAR(20),
            country VARCHAR(50),
            bedrooms FLOAT,
            bathrooms FLOAT,
            homeStatus VARCHAR(50),
            price FLOAT,
            taxAssessedValue FLOAT,
            zestimate FLOAT,
            rentZestimate FLOAT,
            daysOnZillow INTEGER,
            latitude FLOAT,
            longitude FLOAT,
            propertyType VARCHAR(100),
            livingArea FLOAT,
            lotAreaValue FLOAT,
            yearBuilt INTEGER,
            timeOnZillow BIGINT,
            currency VARCHAR(10),
            datePriceChanged BIGINT,
            extracted_date TIMESTAMP,
            created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
            updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
        );
        """)

        success_count = 0
        for i, data in enumerate(transformed_data):
            try:
                cursor.execute("""
                INSERT INTO zillow_properties (
                    zpid, streetAddress, city, state, zipcode, country,
                    bedrooms, bathrooms, homeStatus, price, taxAssessedValue,
                    zestimate, rentZestimate, daysOnZillow, latitude, longitude,
                    propertyType, livingArea, lotAreaValue, yearBuilt,
                    timeOnZillow, currency, datePriceChanged, extracted_date
                ) VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                ON CONFLICT (zpid) DO UPDATE SET
                    streetAddress = EXCLUDED.streetAddress,
                    city = EXCLUDED.city,
                    state = EXCLUDED.state,
                    zipcode = EXCLUDED.zipcode,
                    country = EXCLUDED.country,
                    bedrooms = EXCLUDED.bedrooms,
                    bathrooms = EXCLUDED.bathrooms,
                    homeStatus = EXCLUDED.homeStatus,
                    price = EXCLUDED.price,
                    taxAssessedValue = EXCLUDED.taxAssessedValue,
                    zestimate = EXCLUDED.zestimate,
                    rentZestimate = EXCLUDED.rentZestimate,
                    daysOnZillow = EXCLUDED.daysOnZillow,
                    latitude = EXCLUDED.latitude,
                    longitude = EXCLUDED.longitude,
                    propertyType = EXCLUDED.propertyType,
                    livingArea = EXCLUDED.livingArea,
                    lotAreaValue = EXCLUDED.lotAreaValue,
                    yearBuilt = EXCLUDED.yearBuilt,
                    timeOnZillow = EXCLUDED.timeOnZillow,
                    currency = EXCLUDED.currency,
                    datePriceChanged = EXCLUDED.datePriceChanged,
                    extracted_date = EXCLUDED.extracted_date,
                    updated_at = CURRENT_TIMESTAMP
                """, (
                    data["zpid"], data["streetAddress"], data["city"], data["state"],
                    data["zipcode"], data["country"], data["bedrooms"], data["bathrooms"],
                    data["homeStatus"], data["price"], data["taxAssessedValue"],
                    data["zestimate"], data["rentZestimate"], data["daysOnZillow"],
                    data["latitude"], data["longitude"], data["propertyType"],
                    data["livingArea"], data["lotAreaValue"], data["yearBuilt"],
                    data["timeOnZillow"], data["currency"], data["datePriceChanged"],
                    data["extracted_date"]
                ))
                success_count += 1
            except Exception as e:
                print(f"Error inserting property {i} (zpid: {data.get('zpid')}): {e}")
                continue

        conn.commit()
        cursor.close()
        conn.close()

        print(f"✅ Successfully loaded {success_count} out of {len(transformed_data)} properties into PostgreSQL")

    # Task pipeline
    extracted_data = extract_zillow_data()
    transformed_data = transform_zillow_data(extracted_data)
    load_zillow_data(transformed_data)

```
we will initialize airflow in the following  commands
```
pip install --upgrade apache-airflow  // same thing as    pip install apache-airflow

pip install apache-airflow-providers-postgres
winget install -e --id Astronomer.Astro

astro dev init

astro dev start --verbosity debug

```
we will connect to docker postgres using 
```
psql -U postgres -d postgres
```
