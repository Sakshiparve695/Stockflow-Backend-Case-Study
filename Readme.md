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

---

## 4. Explaination of Fixes

1. Added input validation to prevent crashes
2. Ensured SKU uniqueness to maintain data integrity
3. Used a single transaction to avoid partial data issues
4. Added error handling with rollback
5. Validated price and quantity
6. Improved API response with proper status codes

---

## 4. Assumptions

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
