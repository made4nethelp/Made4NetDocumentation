
# Use DT Connection vs Warehouse Runtime Connection

## Overview

When working with multiple warehouses (multiple database connections), 
the **"Use DT Connection"** checkbox in DT Editor controls which database is used for making inserts, updates, deletes
---

## What Happens When "Use DT Connection" is Checked?

If enabled:

- The DT will always use the database defined in the **Connection** field (e.g., APP).
- It will ignore the warehouse selected at login.
- All inserts and updates will go to that fixed database. This may cause data to be inserted into the wrong database.


---

## What Happens When It Is Unchecked?

If disabled:

- The DT uses the **currently selected warehouse connection**.
- Inserts and updates go to the correct warehouse database.
- DBFIELD expressions execute against the active warehouse.

This is the recommended setup for multiple warehouse environments.

---