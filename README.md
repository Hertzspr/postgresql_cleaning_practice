# postgresql_cleaning_practice
Data cleaning on Nashville Housing in PostgreSQL

This is basically just the SQL code I wrote following Alex Freberg's project tutorial. But he did it in SQL Server while my code is in PostgreSQL. 
What's it like doing it in PostgreSQL? 
+ Easier on Date type initial insertion to the table
- Different treatment of CTE, I couldn't use CTE as a direct reference to delete data, I needed to sort of join it first then pull an identifier to delete the intended rows.

Anyway this is the code:

-- Preview
SELECT * FROM nvl_housing LIMIT 5;

----------------------------------------------------
-- Check Date
SELECT "SaleDate" FROM nvl_housing LIMIT 5;
-- It's already in date format

----------------------------------------------------
-- Populate Property Address Data
-- Preview
SELECT "PropertyAddress" FROM nvl_housing LIMIT 7;

-- Check NULL in Property Address
SELECT "PropertyAddress"
FROM nvl_housing
WHERE "PropertyAddress" IS NULL;

-- Check NULL in the context of the whole row
SELECT * 
FROM nvl_housing
WHERE "PropertyAddress" IS NULL
LIMIT 5;

-- Check ParcelID
SELECT "ParcelID", "PropertyAddress" 
FROM nvl_housing
ORDER BY "ParcelID"
LIMIT 50;

-- Every single ParcelID corresponds to a single PropertyAddress.
-- To repopulate NULL PropertyAddress, we can use ParcelID as a reference and SELF JOIN the table.

-- Preview the solution
SELECT 
    A."UniqueID",
    A."ParcelID",
    A."PropertyAddress",
    B."ParcelID",
    B."PropertyAddress",
    COALESCE(A."PropertyAddress", B."PropertyAddress")
FROM NVL_HOUSING AS A
JOIN NVL_HOUSING AS B ON A."ParcelID" = B."ParcelID"
AND A."UniqueID" != B."UniqueID";

-- Update the table with the solution
UPDATE nvl_housing
SET "PropertyAddress" = COALESCE(nvl_housing."PropertyAddress", B."PropertyAddress")
FROM nvl_housing AS B
WHERE nvl_housing."ParcelID" = B."ParcelID" 
    AND nvl_housing."UniqueID" != B."UniqueID" 
    AND nvl_housing."PropertyAddress" IS NULL;

-- Check whether there is NULL in PropertyAddress or not
SELECT "PropertyAddress"
FROM nvl_housing
WHERE "PropertyAddress" IS NULL;

----------------------------------------------------
-- Break down PropertyAddress into separate columns
SELECT
    SUBSTRING(
        "PropertyAddress"
        FROM 1
        FOR POSITION(',' IN "PropertyAddress") - 1
    ) AS "PropertySplitAddress",
    SUBSTRING(
        "PropertyAddress"
        FROM POSITION(',' IN "PropertyAddress") + 1
        FOR LENGTH("PropertyAddress")
    ) AS "PropertySplitCity"
FROM nvl_housing;

-- Update the table with the new columns
ALTER TABLE nvl_housing 
ADD COLUMN "PropertySplitAddress" VARCHAR(255),
ADD COLUMN "PropertySplitCity" VARCHAR(255);

-- Fill the new columns with data from PropertyAddress
UPDATE nvl_housing
SET "PropertySplitAddress" = SUBSTRING(
        "PropertyAddress"
        FROM 1
        FOR POSITION(',' IN "PropertyAddress") - 1
    ),
    "PropertySplitCity" = SUBSTRING(
        "PropertyAddress"
        FROM POSITION(',' IN "PropertyAddress") + 1
        FOR LENGTH("PropertyAddress")
    );

----------------------------------------------------
-- Break down OwnerAddress
-- Preview the column
SELECT "OwnerAddress"
FROM nvl_housing
LIMIT 10;

-- Use split_part to split the string with a comma delimiter
SELECT
    split_part("OwnerAddress", ',', 1),
    split_part("OwnerAddress", ',', 2),
    split_part("OwnerAddress", ',', 3)
FROM nvl_housing;

-- Update the table with new columns and fill them with split parts
ALTER TABLE nvl_housing
ADD COLUMN "OwnerSplitAddress" VARCHAR(255),
ADD COLUMN "OwnerSplitCity" VARCHAR(255),
ADD COLUMN "OwnerSplitState" VARCHAR(255);

UPDATE nvl_housing
SET 
    "OwnerSplitAddress" = split_part("OwnerAddress", ',', 1),
    "OwnerSplitCity" = split_part("OwnerAddress", ',', 2),
    "OwnerSplitState" = split_part("OwnerAddress", ',', 3);

-- Check the result
SELECT
    "OwnerAddress", "OwnerSplitAddress", "OwnerSplitCity", "OwnerSplitState"
FROM nvl_housing
LIMIT 10;

----------------------------------------------------
-- In "SoldAsVacant" column, there are values which are Y and N that should have been Yes or No
-- Let's check it out
SELECT
    DISTINCT "SoldAsVacant",
    COUNT("SoldAsVacant") AS "SoldAsVacant_Count" 
FROM nvl_housing
GROUP BY "SoldAsVacant"
ORDER BY COUNT("SoldAsVacant") DESC;

-- Use CASE WHEN to change the N to No and Y to Yes
-- Check the validity of the query
SELECT
    "SoldAsVacant",
    CASE 
        WHEN "SoldAsVacant" = 'N' THEN 'No'
        WHEN "SoldAsVacant" = 'Y' THEN 'Yes'
        ELSE "SoldAsVacant" 
    END
FROM nvl_housing
WHERE "SoldAsVacant" IN ('N', 'Y');

-- It's valid
-- Update the table with the CASE WHEN statement
UPDATE nvl_housing
SET "SoldAsVacant" = 
    CASE 
        WHEN "SoldAsVacant" = 'N' THEN 'No'
        WHEN "SoldAsVacant" = 'Y' THEN 'Yes'
        ELSE "SoldAsVacant" 
    END;

-- Check the result again
SELECT
    DISTINCT "SoldAsVacant",
    COUNT("SoldAsVacant") AS "SoldAsVacant_Count" 
FROM nvl_housing
GROUP BY "SoldAsVacant"
ORDER BY COUNT("SoldAsVacant") DESC;

----------------------------------------------------
-- Remove duplicates
-- Create a Common Table Expression (CTE) to identify duplicates
-- The second row with the same items in the partitioned columns is considered a duplicate
WITH row_num_part AS (
    SELECT 
        *,
        ROW_NUMBER() OVER(
        PARTITION BY "ParcelID",
                     "PropertyAddress",
                     "SalePrice",
                     "SaleDate",
                     "LegalReference"
                     ORDER BY "UniqueID"
        ) AS row_num
    FROM nvl_housing
) 

-- Fetch the duplicates
SELECT *
FROM row_num_part
WHERE row_num > 1;

-- Create a temporary table to store the data
DROP TABLE nvl_temp;

CREATE TABLE nvl_temp (LIKE nvl_housing);

INSERT INTO nvl_temp
SELECT * FROM nvl_housing;

-- Delete the duplicates from the temporary table
-- This query joins nvl_temp with row_num_part and uses UniqueID to identify rows to delete
DELETE FROM nvl_temp 
WHERE "UniqueID" in (SELECT "UniqueID" FROM row_num_part WHERE row_num > 1);

-- Check for duplicates in nvl_temp
WITH row_num_part AS (
    SELECT 
        *,
        ROW_NUMBER() OVER(
        PARTITION BY "ParcelID",
                     "PropertyAddress",
                     "SalePrice",
                     "SaleDate",
                     "LegalReference"
                     ORDER BY "UniqueID"
        ) AS row_num
    FROM nvl_temp
) 

SELECT *
FROM row_num_part
WHERE row_num > 1;

-- The temporary table is free from duplicates, so drop the original table and rename the temporary table
DROP TABLE nvl_housing;

ALTER TABLE nvl_temp 
RENAME TO nvl_housing;

-- Check for duplicates in the renamed table
WITH row_num_part AS (
    SELECT 
        *,
        ROW_NUMBER() OVER(
        PARTITION BY "ParcelID",
                     "PropertyAddress",
                     "SalePrice",
                     "SaleDate",
                     "LegalReference"
                     ORDER BY "UniqueID"
        ) AS row_num
    FROM nvl_housing
)

SELECT *
FROM row_num_part
WHERE row_num > 1;

-- No duplicates remain in nvl_housing

----------------------------------------------------
-- Delete unused columns
-- Since we have separated the addresses into more usable columns, delete the original columns
-- We'll delete TaxDistrict as well
ALTER TABLE nvl_housing
DROP COLUMN "PropertyAddress", 
DROP COLUMN "OwnerAddress", 
DROP COLUMN "TaxDistrict";

-- Check the resulting table
SELECT * 
FROM nvl_housing;

