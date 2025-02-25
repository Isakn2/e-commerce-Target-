import kagglehub
import os
import pymysql
import pandas as pd
from zipfile import ZipFile

# Step 1: Download dataset from Kaggle
dataset_path = kagglehub.dataset_download("devarajv88/target-dataset")

print("Dataset downloaded to:", dataset_path)

# Extract ZIP file if needed
for file in os.listdir(dataset_path):
    if file.endswith(".zip"):
        with ZipFile(os.path.join(dataset_path, file), 'r') as zip_ref:
            zip_ref.extractall(dataset_path)
        print(f"Extracted: {file}")

# Step 2: MySQL Database Connection
MYSQL_HOST = "localhost"
MYSQL_USER = "root"
MYSQL_PASSWORD = "your_password"
MYSQL_DB = "EcommerceDB"

connection = pymysql.connect(host=MYSQL_HOST, user=MYSQL_USER, password=MYSQL_PASSWORD, database=MYSQL_DB)
cursor = connection.cursor()

# Step 3: Create tables if they don’t exist
table_queries = {
    "customers": """
        CREATE TABLE IF NOT EXISTS customers (
            customer_id VARCHAR(50) PRIMARY KEY,
            customer_unique_id VARCHAR(50),
            customer_zip_code_prefix INT,
            customer_city VARCHAR(100),
            customer_state VARCHAR(10)
        );
    """,
    "geolocation": """
        CREATE TABLE IF NOT EXISTS geolocation (
            geolocation_zip_code_prefix INT,
            geolocation_lat FLOAT,
            geolocation_lng FLOAT,
            geolocation_city VARCHAR(100),
            geolocation_state VARCHAR(10)
        );
    """,
    "order_items": """
        CREATE TABLE IF NOT EXISTS order_items (
            order_id VARCHAR(50),
            order_item_id INT,
            product_id VARCHAR(50),
            seller_id VARCHAR(50),
            shipping_limit_date DATETIME,
            price FLOAT,
            freight_value FLOAT
        );
    """,
    "orders": """
        CREATE TABLE IF NOT EXISTS orders (
            order_id VARCHAR(50) PRIMARY KEY,
            customer_id VARCHAR(50),
            order_status VARCHAR(20),
            order_purchase_timestamp DATETIME,
            order_approved_at DATETIME,
            order_delivered_carrier_date DATETIME,
            order_delivered_customer_date DATETIME,
            order_estimated_delivery_date DATETIME
        );
    """,
    "payments": """
        CREATE TABLE IF NOT EXISTS payments (
            order_id VARCHAR(50),
            payment_sequential INT,
            payment_type VARCHAR(20),
            payment_installments INT,
            payment_value FLOAT
        );
    """,
    "products": """
        CREATE TABLE IF NOT EXISTS products (
            product_id VARCHAR(50) PRIMARY KEY,
            product_category VARCHAR(100),
            product_name_length INT,
            product_description_length INT,
            product_photos_qty INT,
            product_weight_g INT,
            product_length_cm INT,
            product_height_cm INT,
            product_width_cm INT
        );
    """,
    "sellers": """
        CREATE TABLE IF NOT EXISTS sellers (
            seller_id VARCHAR(50) PRIMARY KEY,
            seller_zip_code_prefix INT,
            seller_city VARCHAR(100),
            seller_state VARCHAR(10)
        );
    """
}

for table, query in table_queries.items():
    cursor.execute(query)
    print(f"Table {table} checked/created.")

# Step 4: Load CSV files into MySQL
def load_csv_to_mysql(table_name, file_path):
    try:
        df = pd.read_csv(file_path)
        df.to_sql(table_name, con=connection, if_exists="append", index=False)
        print(f"Data inserted into {table_name} from {file_path}")
    except Exception as e:
        print(f"Error inserting into {table_name}: {e}")

csv_files = {
    "customers": "customers.csv",
    "geolocation": "geolocation.csv",
    "order_items": "order_items.csv",
    "orders": "orders.csv",
    "payments": "payments.csv",
    "products": "products.csv",
    "sellers": "sellers.csv"
}

for table, file in csv_files.items():
    file_path = os.path.join(dataset_path, file)
    if os.path.exists(file_path):
        load_csv_to_mysql(table, file_path)
    else:
        print(f"File {file_path} not found!")

# Close MySQL connection
cursor.close()
connection.close()
print("Database setup complete! 🚀")
