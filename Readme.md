# StockFlow Backend Engineering Case Study

## Overview
This submission includes solutions for all three parts of the case study. I have focused on clarity, simplicity, and real-world backend practices while making reasonable assumptions where requirements were incomplete.This solution focuses on correctness, simplicity, and handling real-world edge cases.

---
# Part 1: Code Review & Debugging

## 1. Issues Identified

1. No input validation  
The code directly accesses request data without checking if fields are missing.

2. No SKU uniqueness check  
SKU should be unique, but the code does not verify this.

3. Multiple database commits  
Product and inventory are committed separately.

4. No transaction handling  
If one operation fails, there is no rollback.

5. No error handling  
No try-except block is used.

6. Incorrect data modeling  
Product is linked to only one warehouse, but it should support multiple warehouses.

7. No price validation  
Price can be invalid or negative.

8. No quantity validation  
Inventory quantity can be negative.

9. No handling of optional fields  
The code assumes all fields are mandatory.

10. Weak API response  
No proper status codes or structured response.

---

## 2. Impact in Production

- Application may crash if fields are missing  
- Duplicate SKUs can cause data inconsistency  
- Partial data may be saved due to multiple commits  
- Inventory may not match product data  
- Poor user experience due to lack of error handling  
- System becomes difficult to scale and maintain  

---

## 3. Corrected Implementation

```python
@app.route('/api/products', methods=['POST'])
def create_product():
    data = request.json

    try:
        required_fields = ['name', 'sku', 'price', 'warehouse_id', 'initial_quantity']
        for field in required_fields:
            if field not in data:
                return {"error": f"{field} is required"}, 400

        if float(data['price']) < 0:
            return {"error": "Price cannot be negative"}, 400

        if int(data['initial_quantity']) < 0:
            return {"error": "Quantity cannot be negative"}, 400

        existing_product = Product.query.filter_by(sku=data['sku']).first()
        if existing_product:
            return {"error": "SKU already exists"}, 400

        product = Product(
            name=data['name'],
            sku=data['sku'],
            price=float(data['price'])
        )

        db.session.add(product)
        db.session.flush()

        inventory = Inventory(
            product_id=product.id,
            warehouse_id=data['warehouse_id'],
            quantity=data['initial_quantity']
        )

        db.session.add(inventory)

        db.session.commit()

        return {
            "message": "Product created successfully",
            "product_id": product.id
        }, 201

    except Exception as e:
        db.session.rollback()
        return {"error": str(e)}, 500
```
---

## 4. Explaination of Fixes

1. Added input validation to prevent crashes
2. Ensured SKU uniqueness to maintain data integrity
3. Used a single transaction to avoid partial data issues
4. Added error handling with rollback
5. Validated price and quantity
6. Improved API response with proper status codes

---

## 5. Assumptions

1. A product can exist in multiple warehouses
2. Inventory is managed separately from products
3. SKU is globally unique
4. Initial inventory is created when product is added

---

## 6. Questions / Clarifications

1. Can a product exist without being assigned to a warehouse at creation time?
2. Should inventory always be created along with the product, or can it be added/updated later?
3. How should the system handle a product being stored in multiple warehouses?
4. Are there specific validation rules or format constraints for SKU?
5. What fields are optional during product creation?

 # Part 2: Database Design

## Approach
I designed the database keeping in mind real-world use cases like handling multiple warehouses, tracking inventory changes, and managing supplier relationships. I tried to keep the design simple but scalable.

---

## 1. Schema Design

### Companies
- id (Primary Key)
- name (String)
- created_at (Timestamp)

---

### Warehouses
- id (Primary Key)
- name (String)
- location (String)
- company_id (Foreign Key → Companies.id)

---

### Products
- id (Primary Key)
- name (String)
- sku (String, Unique)
- price (Decimal)
- is_bundle (Boolean)
- created_at (Timestamp)

---

### Inventory
- id (Primary Key)
- product_id (Foreign Key → Products.id)
- warehouse_id (Foreign Key → Warehouses.id)
- quantity (Integer)
- last_updated (Timestamp)

---

### Inventory_Logs (to track stock changes)
- id (Primary Key)
- product_id (Foreign Key → Products.id)
- warehouse_id (Foreign Key → Warehouses.id)
- change_in_quantity (Integer)
- updated_at (Timestamp)

---

### Suppliers
- id (Primary Key)
- name (String)
- contact_email (String)

---

### Product_Suppliers (mapping table)
- id (Primary Key)
- product_id (Foreign Key → Products.id)
- supplier_id (Foreign Key → Suppliers.id)

---

### Bundles (for products made of other products)
- id (Primary Key)
- bundle_product_id (Foreign Key → Products.id)
- child_product_id (Foreign Key → Products.id)
- quantity (Integer)

---

## 2. Relationships

- One company can have multiple warehouses  
- One warehouse can store multiple products  
- One product can exist in multiple warehouses  
- One product can have multiple suppliers and vice versa  
- A bundle product can contain multiple other products  

---

## 3. Questions / Missing Requirements

1. Can a product belong to more than one company?  
2. Should inventory updates be real-time or can they be delayed?  
3. Can one supplier provide products to multiple companies?  
4. How should pricing work for bundle products?  

---

## 4. Design Decisions

- I used a separate Inventory table to handle multiple warehouses per product  
- Inventory_Logs table is used to track changes over time  
- SKU is kept unique to avoid duplicate products  
- Used a mapping table for product-supplier relationship  
- Bundle table helps in representing products made of other products  
- I kept the schema simple so it can be extended later if needed  

---

## 5. Assumptions

- Each warehouse belongs to one company  
- A product can exist in multiple warehouses  
- Inventory is tracked per product per warehouse  
- Bundle products are made using existing products  
- Suppliers can provide multiple products

## Part 3: API Implementation

### Approach
I implemented this endpoint by fetching all products for a given company across its warehouses, checking their inventory levels, and comparing them with a defined threshold.  
I also filtered only those products which have recent sales activity and included supplier details for reordering.

---

### Assumptions

- Each product has a predefined low-stock threshold  
- Recent sales activity means sales in the last 30 days  
- Each product has at least one supplier  
- Days until stockout is estimated based on average daily sales  
- Inventory is tracked per product per warehouse  

---

### Implementation (Python + Flask)

```python
@app.route('/api/companies/<int:company_id>/alerts/low-stock', methods=['GET'])
def get_low_stock_alerts(company_id):
    try:
        alerts = []

        # Get all warehouses for the company
        warehouses = Warehouse.query.filter_by(company_id=company_id).all()

        for warehouse in warehouses:
            # Get inventory records for each warehouse
            inventories = Inventory.query.filter_by(warehouse_id=warehouse.id).all()

            for inv in inventories:
                product = Product.query.get(inv.product_id)

                # Assumed threshold (can be based on product type)
                threshold = 20  

                # Skip if stock is sufficient
                if inv.quantity >= threshold:
                    continue

                # Check recent sales (last 30 days)
                recent_sales = Sale.query.filter(
                    Sale.product_id == product.id,
                    Sale.created_at >= datetime.now() - timedelta(days=30)
                ).all()

                if not recent_sales:
                    continue

                # Calculate average daily sales
                total_sold = sum(s.quantity for s in recent_sales)
                avg_daily_sales = total_sold / 30 if total_sold > 0 else 0

                # Estimate days until stockout
                days_until_stockout = int(inv.quantity / avg_daily_sales) if avg_daily_sales > 0 else None

                # Get supplier (first one for simplicity)
                supplier = ProductSupplier.query.filter_by(product_id=product.id).first()
                supplier_data = None

                if supplier:
                    sup = Supplier.query.get(supplier.supplier_id)
                    supplier_data = {
                        "id": sup.id,
                        "name": sup.name,
                        "contact_email": sup.contact_email
                    }

                # Add alert
                alerts.append({
                    "product_id": product.id,
                    "product_name": product.name,
                    "sku": product.sku,
                    "warehouse_id": warehouse.id,
                    "warehouse_name": warehouse.name,
                    "current_stock": inv.quantity,
                    "threshold": threshold,
                    "days_until_stockout": days_until_stockout,
                    "supplier": supplier_data
                })

        return {
            "alerts": alerts,
            "total_alerts": len(alerts)
        }, 200

    except Exception as e:
        return {"error": str(e)}, 500
```
