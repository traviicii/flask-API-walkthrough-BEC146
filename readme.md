# Flask API Setup + Walkthourgh



## 1. Set up coding environment

-  Create a project folder

-  Open a terminal inside project folder

### Create virtual environment

-  Windows:
```bash
python -m venv venv
```
-  Mac:
```bash
python3 -m venv venv
```

### Activate your virtual environment

- Windows:
```bash
venv\Scripts\activate
```
- Mac:
```bash
source venv/bin/activate
```

### Install Flask
Documentation: [Flask Documentation](https://flask.palletsprojects.com/en/3.0.x/)
Quickstart (used in this guide): [Flask Quickstart](https://flask.palletsprojects.com/en/3.0.x/quickstart/)
```bash
pip install flask
```

#### Check for installation and create requirements.txt
*Two seperate commands*
```bash
pip list
pip freeze > requirements.txt
```

### 2. Create `app.py` file

#### Insde `app.py`

```python
# Import the flask class into our project
from flask import Flask

# Create an instance of the Flask class
app = Flask(__name__)


# Routes recieve requests
# Let's define a default "home" route. This will be similar to a landing page.
@app.route('/')
def home():
    return "Welcome to the Flask party" # returning a response

# ensures our app will only run from this file
if __name__ == '__main__':
    app.run(debug = True)
```

### 3. Test run your app

- In the terminal

```bash
flask run
```
Once our Flask server is running, you can visit the link on your localhost to check if everything is running as expected.

---

# Continue building out our app
Continue to create a database connections and API routes that we can use!

## Install flask-marshmallow
Added to your pre-existing imports

```bash
pip install flask-marshmallow
```

## 1. In `app.py`

### Import marshmallow

```python
from flask import Flask
from flask_marshmallow import Marshmallow
from marshmallow import fields
```

### Add an instance of Marshmallow

```python
# Create an instance of the Flask class
app = Flask(__name__)

# create instance of Marshmallow
ma = Marshmallow(app)
```


### Prepare a database to use

### Create a customer table schema for our database

```python
class CustomerSchema(ma.Schema):
    id = fields.Int(dump_only = True) # dump = True, means we don't input data for this field
    customer_name = fields.String(required = True) # required = True, require input for this field
    email = fields.String(required = True)
    phone = fields.String(required = True)

    class Meta:
        fields = ("customer_name", "email", "phone")

customer_schema = CustomerSchema()
customers_schema = CustomerSchema(many = True) # for fetching all customers, many = True allows more than one piece of data
```

## 2. Create a new file `db_connections.py`

### Install `mySQL Connector`

```bash
pip install mysql-connector-python
```

#### Update requirements.txt

```bash
pip freeze > requirements.txt
```



### Create connection function to connect with our database

```python
import mysql.connector
from mysql.connector import Error

# Database connection parameters
db_name = 'ecom_db'
user = 'root'
password = ''
host = 'localhost'

def db_connection():
    try:
        # attempt to establish a connection
        conn = mysql.connector(
            database = db_name,
            user = user,
            password = password,
            host = host
        )
        if conn.is_connected():
            print("Connection to MySQL database successful!")

    except Error as e:
        print(f"Error: {e}")
        return None
            
```

## 3. Build some more routes to add full `CRUD` operations
## Inside of `app.py`

### Update imports
Add these imports: `jsonify` and `db_connection`
```python
from flask import Flask, jsonify
from db_connection import db_connection
```


### Create a new route to `retrieve` all `customers`
This can be placed under the `home` or `'/'` route we created earlier

```python
@app.route('/customers', methods = ['GET'])
def get_customers():
    conn = db_connection()
    if conn is not None:
        try:
            cursor = conn.cursor(dictionary = True) # allows data we get back to be returned as a dictionary to then be transformed into JSON

            # write our SQL query to get all users
            query = "SELECT * FROM Customer"

            cursor.execute(query)

            customers =  cursor.fetchall()

        finally:
            if conn and conn.is_connected():
                cursor.close()
                conn.close()
                return customers_schema.jsonify()
```

### Test your new route!

#### Run your app and navigate to your new API endpoint

- In your terminal, spin up your local server:
```bash
flask run
```


- In your browser
```http
lacalhost:5000/customers
```

## Add Request and Error to imports

```python
from flask import Flask, jsonify, request
from marshmallow import fields, ValidationError
from db_connection import db_connection, Error
```


## Create a route to `create` new customers
### Inside of `app.py`
```python
@app.route('/customers', methods = ['POST'])
def add_customer():
    try:
        customer_data = customer_schema.load(request.json)
        print(customer_data)
    except ValidationErrror as e:
        return jsonify(e.messagges), 400

    conn = db_connection()
    if conn is not None:
        try:
            cursor = conn.cursor()
            
            # new customer details
            new_customer = (customer_data['customer_name'], customer_data['email'], customer_data['phone'])

            # query
            query = "INSERT INTO Customer (customer_name, email, phone) VALUES (%s, %s, %s)"

            # Execute query
            cursor.execute(query, new_customer)
            conn.commit()

            return jsonify({'Message': "New customer added successfully!"}), 201

        except Error as e:
            return jsonify(e.,essages), 500
        finally:
            cursor.close()
            conn.close()

    else:
        return jsonify({"error": "Database connection failed"}), 500
```

### Test the route with Postman

## Create a route to `update` route to update information

```python
@pp.route('/customers/<int:id>', methods = ['PUT'])
def update_customer(id):
    
    try:
        customer_data = customer_schema.load(request.json)
    except ValidationError as e:
        return jsonify(e.messages), 400

    conn = db_connection()
    if conn is not None:
        try:
            cursor = conn.cursor()

            # Checking if the User actually exeists first
            check_query = "SELECT * FROM Customer WHERE id = %s"
            cursor.execute(check_query, (id,))
            customer = cursor.fetchone()
            if not customer:
                return jsonify({"Error": "Customer not found"}), 404

            # Unpacking the updated customer info
            updated_customer = (customer_data['customer_name'], customer_data['email'], customer_data['phone'], id)

            # Update query
            query = "UPDATE Customer SET customer_name = %s, email = %s, phone = %s WHERE id = %s"

            cursor.execute(query, updated_customer)
            conn.commit()

            return jsonify({"Message": f"Successfully udated User {id}"}), 200
        except Error as e:
            return jsonify({"Error": "Internal server error"}), 500
        finally:
            cursor.close()
            conn.close()

    else:
        return jsonify({"Error": "Database connection failed"}), 500
```

### Test the route with Postman

## Create a route to `delete` a customer

```python
@app.route('/customers/<int:id>', methods = ['DELETE'])
def delete_customer(id):
    
    conn = db_connection()
    if conn is not None:
        try:
            cursor = conn.cursor()
            
            # Checking if the User actually exeists first
            check_query = "SELECT * FROM Customer WHERE id = %s"
            cursor.execute(check_query, (id,))
            customer = cursor.fetchone()
            if not customer:
                return jsonify({"Error": "Customer not found"}), 404

            query = "DELETE FROM Customer WHERE id = %s"
            cursor.execute(query, (id,))
            conn.commit()

            return jsonify({"Message": f"Customer {id} was destroyed!"})
        except Error as e:
            return jsonify({"Error": "Internal server error"}), 500
        finally:
            cursor.close()
            conn.close()

    else:
        return jsonify({"Error": "Database connection failed"}), 500

```

### Test the route with Postman
