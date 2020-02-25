# Ship to Shore
#### Mapping the Global Supply Chain with Panjiva Shipping Data in Xpressfeed
### As of Feb 2020; by Richard Torotiello; Temi Oyeniyi, CFA; Zack Yang; Eric Oak

---
**1.** Linking Panjiva Consignees to CIQ Ultimate Parents
Displays the number of US import shipments by consignee for each subsidiary of an ultimate parent (Berkshire Hathaway as an example).
```sql
SELECT c.companyId AS ParentID
     , c.companyName AS ParentName
     , c2.companyId AS SubsidiaryID
     , c2.companyName AS SubsidiaryName
     , conPanjivaId AS ConsigneeID
     , conName AS ConsigneeName
     , COUNT(panjivaRecordId) AS Records
FROM xfl_ciq..ciqCompany c
JOIN xfl_ciq..ciqCompanyUltimateParent cup
-- begin with all CIQ companies
-- link ultimate parent table
ON cup.ultimateParentCompanyId = c.companyId
JOIN xfl_panjiva..panjivaCompanyCrossRef ccr -- Panjiva/CIQ cross ref table
  ON ccr.companyId = cup.companyId
JOIN xfl_ciq..ciqCompany c2
  ON c2.companyId = ccr.companyId
JOIN xfl_panjiva..panjivaUSImport2018 imp
  ON imp.conPanjivaId = ccr.identifierValue
WHERE c.companyId = 255251
GROUP BY c.companyId, c.companyName, c2.companyId , c2.companyName,
  conPanjivaId, conName           -- group by parent, subsidiary, consignee
ORDER BY COUNT(panjivaRecordId) DESC         -- order by record count
```
---
**2.** Time Series Analysis of Shipments by Ultimate Parent
Displays the number of shipment records, TEUs, and metric tons by company over time, for companies in the US auto parts manufacturing industry. Incorporates both US and Mexican import data.
```sql
SELECT c.companyID, c.companyName
  , DATEPART(YEAR,shpmtDate) AS Year_
  , DATEPART(MONTH, shpmtDate) AS Month_
  , COUNT(imp.panjivaRecordId) AS RecordCount
  , SUM(imp.volumeTEU) AS TEUCount
  , SUM(imp.weightT) AS MetricTons
FROM xfl_ciq..ciqCompany c -- begin with all CIQ companies
  JOIN xfl_ciq..ciqCompanyIndustry ci -- link to industry ID table
    ON ci.companyId = c.companyId
  JOIN xfl_ciq..ciqSubTypeToGICS stg -- link to GICs table
    ON stg.subTypeId = ci.industryId
 JOIN xfl_ciq..ciqCompanyUltimateParent cup
   ON cup.ultimateParentCompanyId = c.companyId -- link ultimate parent table
 JOIN xfl_panjiva..panjivaCompanyCrossRef ccr
   ON ccr.companyId = cup.companyId -- link Panjiva/CIQ cross ref table
JOIN( -- join all relevant US Import (top) and Mexican export (bottom) tables 
SELECT conPanjivaID, panjivaRecordId, arrivalDate AS shpmtDate, volumeTEU, weightT
   FROM xfl_panjiva..panjivaUSImport2019
   UNION
 SELECT conPanjivaID, panjivaRecordId, arrivalDate AS shpmtDate, volumeTEU, weightT
   FROM xfl_panjiva..panjivaUSImport2018
   UNION
 SELECT conPanjivaID, panjivaRecordId, arrivalDate AS shpmtDate, volumeTEU, weightT
   FROM xfl_panjiva..panjivaUSImport2017
UNION
SELECT conPanjivaID, panjivaRecordId, shpmtDate, volumeTEU, grossWeightT AS weightT
   FROM xfl_panjiva..panjivaMXExport2019
   WHERE itemDestination = 'United States' and transportMethod != 'Maritime'
   UNION
SELECT conPanjivaID, panjivaRecordId, shpmtDate, volumeTEU, grossWeightT AS weightT 
FROM xfl_panjiva..panjivaMXExport2018
WHERE itemDestination = 'United States' and transportMethod != 'Maritime'
UNION
SELECT conPanjivaID, panjivaRecordId, shpmtDate, volumeTEU, grossWeightT AS weightT 
FROM xfl_panjiva..panjivaMXExport2017
WHERE itemDestination = 'United States' and transportMethod != 'Maritime'
  ) AS imp ON imp.conPanjivaId = ccr.identifierValue
WHERE stg.GIC = 25101010 -- GICS auto parts/eqpmt makers only
  AND c.countryID = 213 -- US Companies Only
  AND shpmtDate >  '2017-08-31' -- Get two full years of data
  AND shpmtDate <= '2019-08-31'
GROUP BY c.companyId, c.companyName, DATEPART(YEAR,shpmtDate),
DATEPART(MONTH,shpmtDate)
ORDER BY 2, 3, 4
```
---
**3.** Health of the Supply Chain: Trends in Supplier Count and Product Count
Displays the number of unique suppliers and unique HS codes for a given ultimate parent on a monthly basis.
```sql
SELECT c.companyId, c.companyName AS CompanyName
     , DATEPART(YEAR, imp.arrivalDate) AS Year_
     , DATEPART(MONTH, imp.arrivalDate) AS Month_
     , COUNT(DISTINCT imp.shpPanjivaId) AS UniqueSuppliers
     , COUNT(DISTINCT spl.value) AS UniqueHSCodes
FROM xfl_ciq..ciqCompany c
JOIN xfl_ciq..ciqCompanyUltimateParent cup ON cup.ultimateParentCompanyId = c.companyId
JOIN xfl_panjiva..panjivaCompanyCrossRef ccr ON ccr.companyId = cup.companyId
JOIN (
                           -- join US import and HS code tables
SELECT conPanjivaId, shpPanjivaId, arrivalDate, hsCode
FROM xfl_panjiva..panjivaUSImport2019 imp2019
JOIN xfl_panjiva..panjivaUSImpHSCode2019 h2019
ON h2019.panjivaRecordId = imp2019.panjivaRecordId
UNION
SELECT conPanjivaId, shpPanjivaId, arrivalDate, hsCode
FROM xfl_panjiva..panjivaUSImport2018 imp2018
JOIN xfl_panjiva..panjivaUSImpHSCode2018 h2018
ON h2018.panjivaRecordId = imp2018.panjivaRecordId
     ) AS imp ON imp.conPanjivaId = ccr.identifierValue
   CROSS APPLY STRING_SPLIT(REPLACE(REPLACE(imp.hscode,'.',''),' ',''),';') spl
    -- evaluate shipments containing multiple HS codes for product analysis
WHERE c.companyID = 266112 -- Deere
GROUP BY c.companyId, c.companyName, DATEPART(YEAR, imp.arrivalDate), DATEPART(MONTH, imp.arrivalDate) -- group by industry, year & month 
ORDER BY 1, 3 DESC, 4 DESC -- order by industry, year & month
```
**4.** Classifying Shipments by GICS Subindustry
Adds industry and GICS tables to display TEU counts and weights by GICS subindustry.
```sql
SELECT stg.GIC, subTypeValue AS Subindustry
     , ROUND(SUM(imp.volumeTEU),0) AS TEUCount
     , ROUND(SUM(imp.weightT),0) AS MetricTons
FROM xfl_ciq..ciqCompany c
JOIN xfl_ciq..ciqCompanyIndustry ci
  ON ci.companyId = c.companyId
JOIN xfl_ciq..ciqSubTypeToGICS stg
  ON stg.subTypeId = ci.industryId
JOIN xfl_ciq..ciqSubType st
  ON st.subTypeId = stg.subTypeId
-- begin with all CIQ companies
-- link to industry ID table
-- link to GICS table
-- link subType for industry name
JOIN xfl_ciq..ciqCompanyUltimateParent cup -- link ultimate parent table
  ON cup.ultimateParentCompanyId = c.companyId
JOIN xfl_panjiva..panjivaCompanyCrossRef ccr -- link Panjiva/CIQ cross ref table
  ON ccr.companyId = cup.companyId
JOIN xfl_panjiva..panjivaUSImport2018 imp -- link 2018 US import table
  ON imp.conPanjivaId = ccr.identifierValue
GROUP BY stg.GIC, subTypeValue -- group by industry
ORDER BY 3 DESC -- order by TEU count
```
---
**5.** Classifying Shipments by HS (Product) Code
Displays the number of shipments that contain a specific product (HS code) and parses shipments that contain multiple codes.
```sql
SELECT spl.value AS HSCode
   , hsc.hsCodeDescription AS HSCodeDescription
   , COUNT(DISTINCT imp.panjivaRecordId) AS Shipments
FROM xfl_ciq..ciqCompany c -- begin with all CIQ companies
JOIN xfl_ciq..ciqCompanyUltimateParent cup -- link ultimate parent table
  ON cup.ultimateParentCompanyId = c.companyId
JOIN xfl_panjiva..panjivaCompanyCrossRef ccr -- Panjiva/CIQ cross ref table
  ON ccr.companyId = cup.companyId
JOIN xfl_panjiva..panjivaUSImport2018 imp -- link 2018 US import table
  ON imp.conPanjivaId = ccr.identifierValue
JOIN xfl_panjiva..panjivaUSImpHSCode2018 h -- link corresp. HS code table
ON h.panjivaRecordId = imp.panjivaRecordId
CROSS APPLY STRING_SPLIT(REPLACE(REPLACE(h.hscode,'.',''),' ',''),';') spl -- splits records with multiple HS codes by separator (semicolon)
              JOIN xfl_panjiva..panjivaHSClassification hsc -- provides HS code text desc
                ON hsc.hsCode = spl.value
              GROUP BY spl.value, hsc.hsCodeDescription -- group by HS Code
              ORDER BY 3 DESC -- order by record count
```
---
6. Time-Series Analysis by Product (HS-6) Code
Displays TEU counts and weight in metric tons by month and year for specific HS codes. The query does not parse records containing multiple HS codes, due to duplicate problems that will affect TEU and weight summarization.
```sql
SELECT LEFT(REPLACE(REPLACE(imp.hscode,'.',''),' ',''),6) AS HSCode
   , DATEPART(YEAR, arrivalDate) AS YEAR_
   , DATEPART(MONTH, arrivalDate) AS MONTH_
   , ROUND(SUM(imp.volumeTEU),0) AS TEUCount
, ROUND(SUM(imp.weightT),0) AS MetricTons
FROM xfl_ciq..ciqCompany c
JOIN xfl_ciq..ciqCompanyUltimateParent cup ON cup.ultimateParentCompanyId = c.companyId
JOIN xfl_panjiva..panjivaCompanyCrossRef ccr ON ccr.companyId = cup.companyId
JOIN (
                -- link 2019 and 2018 US import table with HS code table
SELECT conPanjivaId, volumeTEU, weightT, arrivalDate, hsCode
FROM xfl_panjiva..panjivaUSImport2019 imp2019
JOIN xfl_panjiva..panjivaUSImpHSCode2019 h2019
  ON h2019.panjivaRecordId = imp2019.panjivaRecordId
UNION
SELECT conPanjivaId, volumeTEU, weightT, arrivalDate, hsCode
FROM xfl_panjiva..panjivaUSImport2018 imp2018
JOIN xfl_panjiva..panjivaUSImpHSCode2018 h2018
        ON h2018.panjivaRecordId = imp2018.panjivaRecordId
     ) imp ON imp.conPanjivaId = ccr.identifierValue
WHERE LEFT(REPLACE(REPLACE(imp.hscode,'.',''),' ',''),6)
      IN ('020220', '520512',)                  -- meat cuts, cotton yarn
GROUP BY DATEPART(YEAR, arrivalDate), DATEPART(MONTH, arrivalDate),
  LEFT(REPLACE(REPLACE(imp.hscode,'.',''),' ',''),6) -- group by year, month, and HS code
ORDER BY 1, 2, 3 ASC -- order by record count
```
---
**7.** Country Analysis with US Import Data
Displays the number of shipment records and TEUs imported into the US by country (includes ASEAN member countries as an example).
```sql
SELECT shpmtOrigin AS OriginCountry
     , COUNT(panjivaRecordId) AS RecordCount
     , ROUND(SUM(volumeTEU),0) AS TEUCount
FROM xfl_panjiva..panjivaUSImport2018
WHERE shpmtOrigin IN ('Indonesia', 'Malaysia', 'Philippines', 'Singapore', 'Thailand', 'Vietnam', 'Laos', 'Cambodia', 'Myanmar') -- ASEAN member countries 
GROUP BY shpmtOrigin -- group by origin country 
ORDER BY 2 DESC -- order by record count
```
---
8. Identifying Publicly-Traded Suppliers to a Specific US Industry – by Country
This query identifies ASEAN-country based suppliers to US automotive companies. It employs the ciqSecurity and ciqTradingItem tables to narrow the list to only publicly-traded suppliers.
```sql
SELECT shpmtOrigin, ti2.tradingItemId, ti2.tickerSymbol, c2.companyId,
c2.companyName
  , DATEPART(YEAR, arrivalDate) AS Year_
  , SUM(imp.volumeTEU) AS TEUCount
  , SUM(imp.weightT) AS MetricTons
FROM xfl_ciq..ciqCompany c          -- begin with all CIQ companies
  JOIN xfl_ciq..ciqCompanyIndustry ci
    ON ci.companyId = c.companyId   -- link to industry ID table
  JOIN xfl_ciq..ciqSubTypeToGICS stg
ON stg.subTypeId = ci.industryId -- link to GICs table JOIN xfl_ciq..ciqCompanyUltimateParent cup
    ON cup.ultimateParentCompanyId = c.companyId -- link ultimate parent table
  JOIN xfl_panjiva..panjivaCompanyCrossRef ccr
ON ccr.companyId = cup.companyId -- link consignee to company tables JOIN( -- join all relevant US Import tables
    SELECT conPanjivaId, shpPanjivaID, shpName, panjivaRecordId, arrivalDate,
           volumeTEU, weightT, shpmtOrigin
    FROM xfl_panjiva..panjivaUSImport2019
         ) AS imp ON imp.conPanjivaId = ccr.identifierValue
  JOIN xfl_panjiva..panjivaCompanyCrossRef ccr2
    ON ccr2.identifierValue = imp.shpPanjivaId --link suppliers to cross ref tbl
  JOIN xfl_ciq..ciqCompany c2
  ON c2.companyId = ccr2.companyId          -- link suppliers to company table
  JOIN xfl_ciq..ciqCompanyUltimateParent cup2
ON cup2.ultimateParentCompanyId = c.companyId -- link supp to ult parent JOIN xfl_ciq..ciqSecurity s2 -- link supplier to primary security
ON s2.companyId = c2.companyId and s2.primaryFlag = 1
JOIN xfl_ciq..ciqTradingItem ti2 -- link security to primary ticker ON ti2.securityId = s2.securityId and ti2.primaryFlag = 1
WHERE 1 = 1
AND stg.GIC IN (25101010,25101020,25102010) -- GICS automotive cos.
AND shpmtOrigin IN ('Indonesia', 'Malaysia', 'Philippines', 'Singapore',
'Thailand', 'Vietnam') -- only shipments from ASEAN member countries
  AND datepart(year,arrivalDate) = 2019
  AND ti2.tickerSymbol IS NOT NULL -- supplier must be publicly traded
GROUP BY shpmtOrigin, ti2.tradingItemId, ti2.tickerSymbol, c2.companyId, c2.companyName, datepart(year,arrivalDate) -- group by arrival year and month 
ORDER BY 4,5,6 -- order by year and month
```
---
**9.** Combining Country Datasets: US Imports and Mexican Exports
Adds Mexican rail and truck export data to US maritime import data to get a more complete picture of US imports. Note that field names must be standardized across the two datasets.
```sql
SELECT stg.GIC, st.subTypeValue AS Subindustry
     , DATEPART(YEAR, shpmtDate) AS Year_
     , DATEPART(MONTH, shpmtDate) AS Month_
     , ROUND(SUM(imp.volumeTEU),0) AS TEUCount
FROM xfl_ciq..ciqCompany c
JOIN xfl_ciq..ciqCompanyIndustry ci ON ci.companyId = c.companyId
JOIN xfl_ciq..ciqSubTypeToGICS stg ON stg.subTypeId = ci.industryId
JOIN xfl_ciq..ciqSubType st ON st.subTypeId = stg.subTypeId
JOIN xfl_ciq..ciqCompanyUltimateParent cup ON cup.ultimateParentCompanyId = c.companyId
JOIN xfl_panjiva..panjivaCompanyCrossRef ccr ON ccr.companyId = cup.companyId
shpmtOrigin tradingItemId tickerSymbol companyId companyName
JOIN ( -- join 2019 US import and MX export tables
SELECT conPanjivaID, arrivalDate AS shpmtDate, volumeTEU, weightT
FROM xfl_panjiva..panjivaUSImport2019
UNION
SELECT conPanjivaID, shpmtDate, volumeTEU, grossWeightT AS weightT
FROM xfl_panjiva..panjivaMXExport2019
WHERE itemDestination = 'United States' and transportMethod != 'Maritime' -- only select US shipments that are not shipped by sea
) AS imp ON imp.conPanjivaId = ccr.identifierValue
WHERE stg.GIC IN (20301010, 45202030) -- select GICS subindustries 
GROUP BY stg.GIC, st.subTypeValue, DATEPART(YEAR, shpmtDate), DATEPART(MONTH, shpmtDate) -- group by industry code, industry name, year & month
ORDER BY 1, 3, 4
```
---
Copyright © 2020 by S&P Global Market Intelligence, a division of S&P Global Inc. All rights reserved.
