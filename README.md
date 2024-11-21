# Render-tree
# Generate Item Code with Description

This document provides details for the stored procedure `GenerateItemCodeWithDesc`, the `ItemTbl` table structure, and examples for execution.

## Stored Procedure: `GenerateItemCodeWithDesc`

```
ALTER PROCEDURE GenerateItemCodeWithDesc
    @compId INT,               -- The Unit ID (e.g., "Bd Flour Mills Ltd." or other Unit)
    @itemParentCode NVARCHAR(255),  -- The Parent ITEM_CODE (if it's a child item; NULL for parent items)
    @itemDesc NVARCHAR(255),       -- The Description of the Item
    @createBy INT,                -- User ID of the creator
    @status CHAR(1)               -- Status ('N' for new, 'Y' for active)
AS
BEGIN
    DECLARE @ItemCode NVARCHAR(255);        -- Variable to hold the generated ITEM_CODE
    DECLARE @ParentCode NVARCHAR(255);      -- Parent ITEM_CODE
    DECLARE @ParentItemCode INT;            -- Integer form of the Parent ITEM_CODE
    DECLARE @ChildSequence INT;             -- Sequential number for child items
    DECLARE @SubChildSequence INT;          -- Sequential number for sub-child items

    -- If the item is a parent, generate ITEM_CODE as a simple number (e.g., 101, 102, 103)
    IF @itemParentCode IS NULL
    BEGIN
        -- Generate ITEM_CODE for parent items
        SELECT @ItemCode = CAST(ISNULL(MAX(CAST(ITEM_CODE AS INT)), 100) + 1 AS NVARCHAR(255))
        FROM ItemTbl
        WHERE COMP_ID = @compId AND ITEM_PARENT_CODE = CAST(@compId AS NVARCHAR(255));  -- Root parent items

        IF CAST(@ItemCode AS INT) <= 100
            SET @ItemCode = '101';

        SET @ParentCode = CAST(@compId AS NVARCHAR(255));
    END
    ELSE
    BEGIN
        -- Generate ITEM_CODE for child or sub-child items
        SELECT @ParentItemCode = CAST(ITEM_CODE AS INT)
        FROM ItemTbl
        WHERE ITEM_CODE = @itemParentCode AND COMP_ID = @compId;

        IF LEN(@itemParentCode) = 3
        BEGIN
            SELECT @ChildSequence = ISNULL(MAX(CAST(SUBSTRING(ITEM_CODE, LEN(@itemParentCode) + 1, 3) AS INT)), 0) + 1
            FROM ItemTbl
            WHERE ITEM_PARENT_CODE = @itemParentCode;

            SET @ItemCode = @itemParentCode + RIGHT('000' + CAST(@ChildSequence AS NVARCHAR(3)), 3);
            SET @ParentCode = @itemParentCode;
        END
        ELSE
        BEGIN
            SELECT @SubChildSequence = ISNULL(MAX(CAST(SUBSTRING(ITEM_CODE, LEN(@itemParentCode) + 1, 3) AS INT)), 0) + 1
            FROM ItemTbl
            WHERE ITEM_PARENT_CODE = @itemParentCode;

            SET @ItemCode = @itemParentCode + RIGHT('000' + CAST(@SubChildSequence AS NVARCHAR(3)), 3);
            SET @ParentCode = @itemParentCode;
        END
    END

    -- Concatenate the ITEM_DESC with ITEM_CODE in the desired format
    SET @itemDesc = @itemDesc +' ' + '[' + @ItemCode + ']';

    -- Insert the new item into the Items table
    INSERT INTO ItemTbl (COMP_ID, ITEM_CODE, ITEM_DESC, ITEM_PARENT_CODE, CREATE_BY, STATUS)
    VALUES (@compId, @ItemCode, @itemDesc, @ParentCode, @createBy, @status);
    
    -- Optionally, return the new item ID or ITEM_CODE
    SELECT SCOPE_IDENTITY() AS NewItemID, @ItemCode AS NewItemCode;
END;
```

 
This procedure generates unique item codes and descriptions for a hierarchical item structure, supporting parent, child, and sub-child items.

### **Parameters**
| Parameter          | Data Type         | Description                                                                 |
|--------------------|-------------------|-----------------------------------------------------------------------------|
| `@compId`          | `INT`            | Unit ID (e.g., "Bd Flour Mills Ltd.")                                     |
| `@itemParentCode`  | `NVARCHAR(255)`  | Parent item code (`NULL` for parent items)                                  |
| `@itemDesc`        | `NVARCHAR(255)`  | Item description                                                            |
| `@createBy`        | `INT`            | User ID of the creator                                                      |
| `@status`          | `CHAR(1)`        | Item status (`'N'` for new, `'Y'` for active, etc.)                         |

---

## Table Structure: `ItemTbl`

### **Schema**
```sql
CREATE TABLE ItemTbl (
    ITEM_ID INT PRIMARY KEY IDENTITY(1,1),    -- Auto-incremented primary key
    COMP_ID INT,                              -- Company/Unit ID
    ITEM_CODE NVARCHAR(255),                  -- Unique item code
    ITEM_DESC NVARCHAR(255),                  -- Item description
    ITEM_PARENT_CODE NVARCHAR(255),           -- Parent item code (if any)
    CREATE_BY INT,                            -- User ID of the creator
    STATUS CHAR(1),                           -- Item status ('N', 'Y', etc.)
    DATE_CREATED DATETIME DEFAULT GETDATE()   -- Timestamp of creation
);
```

---

## Execution Examples

### **Insert a Parent Item**
```sql
EXEC GenerateItemCodeWithDesc
    @compId = 53,               -- Unit ID
    @itemParentCode = NULL,     -- No parent code (parent item)
    @itemDesc = 'School Asset ',-- Item description
    @createBy = 1,              -- User ID
    @status = 'N';              -- Status: 'N' (new)
```

### **Insert a Child Item**
```sql
EXEC GenerateItemCodeWithDesc
    @compId = 53,               -- Unit ID
    @itemParentCode = '102',    -- Parent item code
    @itemDesc = 'Chair',        -- Item description
    @createBy = 1,              -- User ID
    @status = 'N';              -- Status: 'N' (new)
```

---

## Features

- **Parent Item Code Generation**: Automatically creates unique parent item codes when `@itemParentCode` is `NULL`.
- **Child and Sub-Child Code Generation**: Hierarchical code generation for items based on their parent item code.
- **Descriptive Codes**: Concatenates item description with the generated item code for clarity.

---

## View Items

Retrieve all items using the following query:
```sql
SELECT * FROM ItemTbl;
```

---

## Notes

- Ensure the `ItemTbl` table exists before executing the procedure.
- Use proper error handling and validation mechanisms for production environments.
