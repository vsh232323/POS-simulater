import sqlite3
import re

# Initializing SQLite3 database
conn = sqlite3.connect('pos_database.db')
cursor = conn.cursor()

# Creating Products table with product id as primary key
cursor.execute('''
    CREATE TABLE IF NOT EXISTS Products (
        id INTEGER PRIMARY KEY,
        name TEXT NOT NULL,
        price REAL NOT NULL,
        quantity INTEGER NOT NULL,
        expiryDate Date,
        ManufacturingDate Date
    )
''')
conn.commit()
productDetails = [
    ('Apple', 150, 100, '2024-01-31', '2024-1-01'),
    ('Banana', 25, 100, '2024-01-15', '2024-1-02'),
    ('Orange', 80, 100, '2024-01-31', '2024-1-01'),
    ('Laptop', 100000.00, 20, None, None),
    ('Headphones', 5000.00, 20, None, None),
    ('Peanut butter', 499.00, 50, '2024-06-6', '2024-01-01'),
    ('Protein Powder', 4999.00, 50, '2024-06-6', '2023-11-01'),
    ('Bottle', 199.00, 50, '2025-01-01', '2024-01-22')
]

cursor.executemany('INSERT INTO Products (name, price, quantity, expiryDate, ManufacturingDate) VALUES (?, ?, ?, ?, ?)',productDetails)
conn.commit()

class ItemsCart:
    def _init_(self):
        self.items = []

    def addItem(self, productId, quantity):
        self.items.append({'productId': productId, 'quantity': quantity})

    def cartSubtotal(self):
        subtotal = 0
        for item in self.items:
            productId = item['productId']
            quantity = item['quantity']
            cursor.execute('SELECT price FROM Products WHERE id = ?', (productId,))
            price = cursor.fetchone()[0]
            subtotal += price * quantity
        return subtotal

    def cartTotal(self):
        total = 0
        for item in self.items:
            productId = item['productId']
            quantity = item['quantity']
            cursor.execute('SELECT price FROM Products WHERE id = ?', (productId,))
            price = cursor.fetchone()[0]
            total += price * quantity
        return total

def display_product_details(product_name, product_price, quantity, expiry_date, manufacturing_date, **kwargs):
    print(f"Product: {product_name}, Price: ₹{product_price:.2f}, Quantity: {quantity}", end=' ')

    if expiry_date:
        print(f"Expiry Date: {expiry_date}")

    if manufacturing_date:
        print(f"Manufacturing Date: {manufacturing_date}")

    if 'discount' in kwargs:
        print(f"Discount: {kwargs['discount']:.2f}%")

    if 'total' in kwargs:
        print(f"Total: ₹{kwargs['total']:.2f}")

    print()

# Lambda function
calculate_discounted_price = lambda price, discount: price - (price * discount / 100)

# List comprehension
discounted_prices = [calculate_discounted_price(price, 10) for price in [20, 30, 40]]

# Exception handling
def quantityValidation(quantity):
    try:
        quantity = int(quantity)
        if quantity <= 0:
            raise ValueError("Quantity must be greater than zero.")
        return quantity
    except ValueError as e:
        print(f"Error: {e}")
        return 0

# Iterators
def productsIterate():
    cursor.execute('SELECT * FROM Products')
    for row in cursor.fetchall():
        yield row

def productNameValidation(product_name):
    pattern = re.compile(r'^[a-zA-Z0-9\s!"#$%&\'()*+,-./:;<=>?@[\\]^_`{|}~]+$')
    return bool(pattern.match(product_name))

class productDiscount(ItemsCart):
    def _init_(self, discount):
        super()._init_()
        self.discount = discount

    def cartTotal_with_discount(self):
        total = super().cartTotal()
        return total - (total * self.discount / 100)

class POSsimulator:
    def _init_(self):
        self.cart = productDiscount(discount=10)

    def show_menu(self):
        while True:
            print("\nMenu:")
            print("1. Insert New Item")
            print("2. View List of Items")
            print("3. Insert Items to Cart")
            print("4. View Cart")
            print("5. Delete Item from Database")
            print("6. Finish and Calculate Total")
            choice = input("Enter your choice (1/2/3/4/5/6): ")

            if choice == '1':
                self.itemInsert()
            elif choice == '2':
                self.itemView()
            elif choice == '3':
                self.itemInsertCart()
            elif choice == '4':
                self.viewCart()
            elif choice == '5':
                self.itemDelete()
            elif choice == '6':
                break
            else:
                print("Invalid choice. Please enter a valid option.")

    def itemInsert(self):
        print("\nInsert New Item")
        product_name = input("Enter product name: ")
        product_price = float(input("Enter product price: "))
        quantity = quantityValidation(input("Enter quantity: "))
        expiry_date = input("Enter expiry date (YYYY-MM-DD): ")
        manufacturing_date = input("Enter manufacturing date (YYYY-MM-DD): ")

        # if not productNameValidation(product_name):
        #     print("Invalid product name. Please enter a valid name.")
        #     return

        cursor.execute('INSERT INTO Products (name, price, quantity, expiryDate, ManufacturingDate) VALUES (?, ?, ?, ?, ?)',(product_name, product_price, quantity, expiry_date, manufacturing_date))
        conn.commit()
        print(f"Item '{product_name}' added to the database successfully.")

    def itemView(self):
        print("\nList of Items:")
        for product in productsIterate():
            productId, product_name, product_price, quantity, expiry_date, manufacturing_date = product
            print(f"{productId}. {product_name} - ₹{product_price:.2f} (Quantity: {quantity})")

    def itemInsertCart(self):
        print("\nInsert Items to Cart:")
        while True:
            productId = int(input("Enter the product ID (0 to finish): "))
            if productId == 0:
                break
            if not any(product[0] == productId for product in productsIterate()):
                print(f"Product with ID {productId} does not exist in the database. Please enter a valid product ID.")
                continue

            quantity = quantityValidation(input("Enter quantity: "))
            if quantity == 0:
                print("Invalid quantity. Please try again.")
                continue
            cursor.execute('SELECT quantity FROM Products WHERE id = ?', (productId,))
            available_quantity = cursor.fetchone()[0]
            if quantity > available_quantity:
                print(f"Insufficient quantity available for product ID {productId}. Available quantity: {available_quantity}")
                continue

            self.cart.addItem(productId, quantity)
            print(f"{quantity} units of product with ID {productId} added to the cart.")


    def viewCart(self):
        print("\nShopping Cart:")
        product_details_dict = {product[0]: product for product in productsIterate()}

        for item in self.cart.items:
            productId, quantity = item['productId'], item['quantity']
            product_details = product_details_dict.get(productId)
            if product_details:
                item_total = product_details[2] * quantity 
                display_product_details(product_details[1], product_details[2], quantity, product_details[4], product_details[5], total=item_total, discount=10)

    def itemDelete(self):
        print("\nDelete Item from Database:")
        productId = int(input("Enter the product ID to delete: "))
        cursor.execute('DELETE FROM Products WHERE id = ?', (productId,))
        conn.commit()
        print(f"Item with ID {productId} deleted from the database.")

    def run(self):
        self.show_menu()
        subtotal = self.cart.cartSubtotal()
        total_with_discount = self.cart.cartTotal_with_discount()

        print("\nShopping Cart:")
        product_details_dict = {product[0]: product for product in productsIterate()}

        for item in self.cart.items:
            productId, quantity = item['productId'], item['quantity']
            product_details = product_details_dict.get(productId)
            if product_details:
                item_total = product_details[2] * quantity 
                display_product_details(product_details[1], product_details[2], quantity, product_details[4], product_details[5], total=item_total, discount=10)

        print(f"Subtotal: ₹{subtotal:.2f}")
        print(f"Total with Discount for all items: ₹{total_with_discount:.2f}")

# Instantiate the POSsimulator and run it
pos_system = POSsimulator()
pos_system.run()

conn.close()
