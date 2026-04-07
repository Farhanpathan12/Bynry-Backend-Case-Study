Bynry Case Study: StockFlow Inventory Management
Role: Backend Engineering Intern Candidate: Farhan Pathan

Part 1: Code Review & Debugging (30 minutes)
1. Identified Issues
Based on the provided Flask API endpoint, here are the critical technical and business logic flaws:
Lack of Database Transaction (Atomicity): The code calls db.session.commit() twice. If the Product creation succeeds but the Inventory creation fails, the database is left in an inconsistent state (a product with no initial inventory).
Data Model Flaw (warehouse_id in Product): The requirements explicitly state "Products can exist in multiple warehouses". Storing warehouse_id directly inside the Product model implies a 1-to-Many relationship, which breaks the business logic. It should only exist in the Inventory junction table.
Missing Error Handling & Validation: There is no try-except block to catch database errors. Furthermore, using data['key'] for optional fields will throw a KeyError and crash the server with a 500 Internal Server Error if the payload omits them.
Missing SKU Uniqueness Check: The system requires SKUs to be unique, but the code blindly inserts data without checking if the SKU already exists.
2. Impact in Production
Data Corruption: "Orphaned" products without inventory records will clutter the database and cause issues in reporting and frontend UI.
System Crashes: Missing payload keys will lead to unhandled exceptions, degrading user experience.
Business Logic Failure: The system will fail to allow the same product to be stocked in multiple warehouses because it's hardcoded to a single warehouse at the product level.
3. Provided Fix
Here is the refactored, production-ready code:
Python
@app.route('/api/products', methods=['POST'])
def create_product():
    data = request.json
    
    # 1. Basic Validation
    if not data or not data.get('name') or not data.get('sku'):
        return {"error": "Name and SKU are required fields"}, 400

    try:
        # 2. Check for SKU Uniqueness
        if Product.query.filter_by(sku=data['sku']).first():
            return {"error": f"Product with SKU '{data['sku']}' already exists"}, 409

        # 3. Create Product (Removed warehouse_id from base model)
        product = Product(
            name=data['name'],
            sku=data['sku'],
            price=float(data.get('price', 0.0)) # Handle optional decimal
        )
        db.session.add(product)
        db.session.flush() # Flush to generate product.id without committing

        # 4. Update Inventory if warehouse data is provided
        warehouse_id = data.get('warehouse_id')
        initial_qty = data.get('initial_quantity')

        if warehouse_id and initial_qty is not None:
            inventory = Inventory(
                product_id=product.id,
                warehouse_id=warehouse_id,
                quantity=int(initial_qty)
            )
            db.session.add(inventory)

        # 5. Single commit for atomicity (All or Nothing)
        db.session.commit()
        return {"message": "Product created successfully", "product_id": product.id}, 201

    except Exception as e:
        db.session.rollback() # Revert all changes if anything fails
        # Log error here in a real production system
        return {"error": "An internal server error occurred"}, 500


Part 2: Database Design (25 minutes)
1. Schema Design (SQL DDL Notation)
SQL
-- Core Entities
CREATE TABLE companies (
    id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL
);

CREATE TABLE warehouses (
    id UUID PRIMARY KEY,
    company_id UUID REFERENCES companies(id),
    name VARCHAR(255) NOT NULL
);

CREATE TABLE suppliers (
    id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    contact_email VARCHAR(255) NOT NULL
);

-- Products & Bundles
CREATE TABLE products (
    id UUID PRIMARY KEY,
    sku VARCHAR(100) UNIQUE NOT NULL,
    name VARCHAR(255) NOT NULL,
    price DECIMAL(10, 2) NOT NULL,
    product_type VARCHAR(100),
    supplier_id UUID REFERENCES suppliers(id),
    is_bundle BOOLEAN DEFAULT FALSE
);

CREATE TABLE product_bundles (
    bundle_product_id UUID REFERENCES products(id),
    component_product_id UUID REFERENCES products(id),
    quantity_required INT NOT NULL,
    PRIMARY KEY (bundle_product_id, component_product_id)
);

-- Inventory & Tracking
CREATE TABLE inventory (
    id UUID PRIMARY KEY,
    product_id UUID REFERENCES products(id),
    warehouse_id UUID REFERENCES warehouses(id),
    quantity INT DEFAULT 0,
    low_stock_threshold INT DEFAULT 10,
    last_sold_date TIMESTAMP,
    avg_daily_sales DECIMAL(10,2) DEFAULT 0.0,
    UNIQUE(product_id, warehouse_id) -- Prevent duplicate tracking rows
);

CREATE TABLE inventory_logs (
    id UUID PRIMARY KEY,
    inventory_id UUID REFERENCES inventory(id),
    change_amount INT NOT NULL, -- e.g., +50 (restock) or -5 (sale)
    reason VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

2. Missing Requirements (Questions for Product Team)
Bundle Logistics: When a bundle is sold, does it deduct from a pre-assembled "bundle inventory", or does it dynamically deduct the individual components from the general stock?
Supplier Mapping: Can a single product be sourced from multiple suppliers? The current assumption is 1 product = 1 supplier.
Log Retention: How long should we keep data in the inventory_logs table before archiving it? High-volume systems will bloat this table quickly.
3. Decisions & Justifications
Junction Table (inventory): Designed to handle the Many-to-Many relationship between products and warehouses, which is the core requirement.
Audit Trail (inventory_logs): Added to satisfy the "Track when inventory levels change" requirement without constantly overwriting historical data.
Indexes: Added a UNIQUE constraint on (product_id, warehouse_id) to ensure data integrity and speed up lookups during API calls.

Part 3: API Implementation (35 minutes)
1. Implementation (Node.js / Express with SQL Query Builder)
Language/Framework: Node.js with Express.js.
JavaScript
const express = require('express');
const router = express.Router();
const db = require('../db'); // Hypothetical DB connection (e.g., Knex.js)

/**
 * GET /api/companies/:company_id/alerts/low-stock
 */
router.get('/api/companies/:company_id/alerts/low-stock', async (req, res) => {
    try {
        const { company_id } = req.params;
        const RECENT_SALES_DAYS = 30; // Business Rule: "Recent" defined as 30 days

        // Fetching data joining inventory, warehouses, products, and suppliers
        const alertsData = await db('inventory as i')
            .join('warehouses as w', 'i.warehouse_id', 'w.id')
            .join('products as p', 'i.product_id', 'p.id')
            .leftJoin('suppliers as s', 'p.supplier_id', 's.id')
            .where('w.company_id', company_id)
            // Rule: Only alert for products with recent sales activity
            .andWhere('i.last_sold_date', '>=', db.raw(`NOW() - INTERVAL '${RECENT_SALES_DAYS} days'`))
            // Rule: Low stock check based on dynamic threshold
            .andWhereRaw('i.quantity <= i.low_stock_threshold')
            .select(
                'p.id as product_id',
                'p.name as product_name',
                'p.sku',
                'w.id as warehouse_id',
                'w.name as warehouse_name',
                'i.quantity as current_stock',
                'i.low_stock_threshold as threshold',
                'i.avg_daily_sales',
                's.id as supplier_id',
                's.name as supplier_name',
                's.contact_email'
            );

        // Map data to the expected JSON format
        const formattedAlerts = alertsData.map(row => {
            // Calculate days until stockout safely
            let daysUntilStockout = 0;
            if (row.avg_daily_sales > 0) {
                daysUntilStockout = Math.max(0, Math.round(row.current_stock / row.avg_daily_sales));
            }

            return {
                product_id: row.product_id,
                product_name: row.product_name,
                sku: row.sku,
                warehouse_id: row.warehouse_id,
                warehouse_name: row.warehouse_name,
                current_stock: row.current_stock,
                threshold: row.threshold,
                days_until_stockout: daysUntilStockout,
                supplier: row.supplier_id ? {
                    id: row.supplier_id,
                    name: row.supplier_name,
                    contact_email: row.contact_email
                } : null // Handle case where a product has no supplier yet
            };
        });

        return res.status(200).json({
            alerts: formattedAlerts,
            total_alerts: formattedAlerts.length
        });

    } catch (error) {
        console.error('Error generating low stock alerts:', error);
        return res.status(500).json({ error: 'Internal Server Error' });
    }
});

module.exports = router;

2. Edge Cases Handled
Division by Zero: The days_until_stockout calculation handles cases where avg_daily_sales is 0 or null to prevent server crashes or returning NaN.
Missing Suppliers: Used a LEFT JOIN for suppliers. If a product doesn't have a supplier assigned, the API won't drop the row; it will simply return null for the supplier object.
Negative Stockout Days: Used Math.max(0, ...) to ensure we don't return negative days if the inventory somehow drops below zero.
3. Documented Assumptions
Recent Activity Definition: I assumed "recent sales activity" means sales within the last 30 days.
Pre-calculated Analytics: Calculating avg_daily_sales dynamically on every API call is computationally expensive. I assumed there is a background Cron Job that calculates and updates avg_daily_sales on the inventory table periodically to keep this endpoint highly performant.
Dynamic Threshold Location: I placed the low_stock_threshold on the inventory table rather than the products table, assuming that different warehouses might have different threshold needs for the exact same product.

