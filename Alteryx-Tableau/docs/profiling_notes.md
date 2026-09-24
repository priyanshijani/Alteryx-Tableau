# Profiling Notes
*Write what you find while profiling (Phase 1). This becomes evidence of your analytical process.*

| # | File | Field | What looks wrong | Example value | How I'll fix it |
|---|---|---|---|---|---|
| 1 | accounts_export |Country| Country grouping incorrect | Germany has value : "GER","Germany"; Franch has :"FR", "FRANCE", etc |  I'll all the fuzzy matches and group them together and rename them in standardized format for proper dimensional grouping on the dashboard
|2 |Opportunities_YYYY_MM|Sales_Stage|The grouping is improper |stage 1 looks: "Stage 1: Discovery (0-50%)","Discovery"|fuzzy match or text containing workflow to standardized names in this column| 
|3 |Opportunities_YYYY_MM||confidence levels are not present explicitly as a data field|0%,50%,75%,etc in a explicit column |I will utilised cleaned Sales_Stage column to derive| 
|4 |Opportunities_YYYY_MM||Explicit stage status is not present if it is active/lost/stale ||Put in a formula logic to check if the deal has been jumping stages using Last_Activity date and first data this opportunity was introduced in the pipeline, first date it was introduced in the current confidence stage | 
|5 |Opportunities_YYYY_MM|Deal_Currency|the currency names are inconsistent and have null values |euro currency has values: "EUR","eur"|use uppercase function to group the currencies together, handle null currency columns in some way|
|6|Opportunities_YYYY_MM|Deal_Value|Values have null values, some have commas, decimals, space trails, alphabets|"9,200.00","1 506 600", " ", "44,500.00", "PLN 331,000"|use regex cleaning to correct number format and change data type to numerical to perform mathematical additions, etc functions for analysis|
|7|Opportunities_YYYY_MM|Deal_Value|Each Deal has a set of currency so total pipeline value in single currency is difficult to measure on dashboard | | we use a standardized currency and convert it to a single currency (Euro) to understand each deal's value in the same currency|
|8|Opportunities_YYYY_MM|Owner_Email|the table has email id of the lead owner and not employee ID to map with Sales team roaster file to extract lead owner related data via join||We have to be careful, perhaps standardize the Owner_Email column to avoid not matching with Sales Leads roaster file so we can extract all related data from dimensional table or we can extract first& last name to match to get employee id into Opportunities_YYYY_MM table and then join Sales lead roaster dimension table using employee IDs|
|9|Accounts_export|Industry|incorrect segmentation for proper grouping|Mixed values like : "Manufacturing", "Manufacturing and Automative", "Automative"|create a new industry grouping logic to correctly group |

