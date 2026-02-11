# onpropertychange Event

## Overview

To go through onpropertychange take an example of purchase order screen from BE client and see how the **PACK** and **PO Current Cost
(UNITCOST)** fields are automatically calculated when the **SKU** field
changes in the Purchase Order Details screen.

DT Used: `DT_PODETAIL_Insert`\
Table Used: `vBE_POSKUPACK`

Event Trigger:

    SKU:onpropertychange

------------------------------------------------------------------------

# 1. PACK Field Expression

## Expression (DT Editor)

    [0];DBFIELD;[1];DBFIELD;[2];
    DBFIELD:(CASE_PACK,vBE_POSKUPACK,
    SKU = '{0}' AND POID = '{1}' AND CONSIGNEE = '{2}'
    ~FIELD:SKU~FIELD:POID~FIELD:CONSIGNEE,)

## Meaning

When SKU changes:

1.  System reads:
    -   SKU (FIELD:SKU)
    -   POID (FIELD:POID)
    -   CONSIGNEE (FIELD:CONSIGNEE)
2.  It replaces placeholders:
    -   `{0}` → SKU
    -   `{1}` → POID
    -   `{2}` → CONSIGNEE
3.  Executes equivalent query:

``` sql
SELECT CASE_PACK
FROM vBE_POSKUPACK
WHERE SKU = '<SKU>'
  AND POID = '<POID>'
  AND CONSIGNEE = '<CONSIGNEE>';
```

4.  Returned value populates **PACK** field.

------------------------------------------------------------------------

# 2. PO Current Cost (UNITCOST) Expression

## Expression

    [0];DBFIELD;[1];DBFIELD;[2];
    DBFIELD:(TOTCOST,vBE_POSKUDEFAULTCOST,
    SKU = '{0}' AND POID = '{1}' AND CONSIGNEE = '{2}'
    ~FIELD:SKU~FIELD:POID~FIELD:CONSIGNEE,)

## Meaning

When SKU changes:

1.  System reads SKU, POID, CONSIGNEE.
2.  Replaces placeholders.
3.  Executes equivalent query:

``` sql
SELECT TOTCOST
FROM vBE_POSKUDEFAULTCOST
WHERE SKU = '<SKU>'
  AND POID = '<POID>'
  AND CONSIGNEE = '<CONSIGNEE>';
```

4.  Returned value populates **PO Current Cost (UNITCOST)**.

------------------------------------------------------------------------

# 3. Execution Flow

    User changes SKU
            ↓
    SKU:onpropertychange event fires
            ↓
    PACK expression executes (DBFIELD)
            ↓
    UNITCOST expression executes (DBFIELD)
            ↓
    Values auto-populated in UI

------------------------------------------------------------------------

## 4. Tabular form of Fields returned from Views

| Field     | Source View              | Column Returned | Trigger                 |
|-----------|--------------------------|-----------------|-------------------------|
| PACK      | vBE_POSKUPACK            | CASE_PACK       | SKU:onpropertychange    |
| UNITCOST  | vBE_POSKUDEFAULTCOST     | TOTCOST         | SKU:onpropertychange    |

------------------------------------------------------------------------

In this way we can populate fields automatically through field expression.
