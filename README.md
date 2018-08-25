# Analytics Assignment

**Author:** Nicholas Scherrer
**Date:** 8/24/2018
* * *
## My approach... 
...to solving these three problems - started with questions fairly unrelated to databases, SQL queries, or even data itself. Rather, the questions required a mix of healthcare domain knowledge, some critical thinking, and lastly the willingness to make sound judgement calls that can be logically defended in front of an audience. A particular roadblock needed to be passed before answering any of the three questions.
 
I needed to know how a ___Primary Care Practice___ was technically defined, as the
supplied data contained no indications or flags in any way. The research I peered into on the internet was of surprisingly little help; varying opinions on what makes a practice fit this classification, along with little to no documentation on the matter from any typically reliable resource made this an issue without a concrete answer which would require assumptions to be made. 

Although most are familiar with Primary Care Physicians conceptually, this one bit of knowledge helped guide the rest of my approach.

> “A **primary care physician (PCP)** is a physician who provides both the first contact for a person with an undiagnosed health concern as well as continuing care of varied medical conditions, **not limited by cause, organ system, or diagnosis**.” _–www.med.uottawa.ca_

Now that my general, initial approach to solving this problem is understood, here's the nitty gritty.
* * *

0. **What the data gave me:**
  - Each providers’ specialty
  - What groups/HC organizations a provider belonged to

This was enough for me to bridge the gap; as shown in the following functional steps, with relevant accompanying code:

1. **Understanding the fundamental role** of a primary care physician allowed me to group them based on their specialization(s). The five I chose were; _Family Practice, Internal Medicine, Pediatrics, OB/GYN,_ and _General Practice_.

``` sql
ALTER TABLE [dbo].[physician_compare_table] ADD  [Primary_Care_Provider_Flag] AS CAST (
CASE 
	WHEN (
   			   [primary_specialty] = 'INTERNAL MEDICINE'
			OR [primary_specialty] = 'FAMILY PRACTICE'
			OR [primary_specialty] = 'OBSTETRICS & GYNECOLOGY'
			OR [primary_specialty] = 'PEDIATRIC MEDICINE'
			OR [primary_specialty] = 'GENERAL PRACTICE'
		)
		THEN 'Yes'
	ELSE 'No'
END AS VARCHAR(50)
	)
```

2. **Armed with** a newly filtered list of providers, I needed to find the most efficient way to determine the classifications of roughly 400 practices. I first noticed specialty practices in the ___Compare Physicians___ dataset; many of which did not have one of these 5 providers on staff. I joined NPI data with their corresponding practices and was quickly able to filter out any practice with zero providers meeting any of my 5 criteria. But this was just low-hanging fruit. I wouldn’t be able to manually maintain that process once the numbers got closer.

``` sql
SELECT comp.organization_legal_name AS Practice_Name
	,COUNT(DISTINCT comp.npi) AS Total_Providers
	,COUNT(DISTINCT a.npi) AS Total_Primary_Care_Providers
INTO Practice_Pivot_Table ---creating a new table
FROM [physician_compare_table] AS comp
LEFT JOIN [physician_compare_table] AS a ON a.npi = comp.npi
	AND a.Primary_Care_Provider_Flag = 'Yes' ---joining data
GROUP BY comp.organization_legal_name
ORDER BY COUNT(DISTINCT comp.npi) DESC
```
| **Practice Name**                        	| **Total Providers** 	| **Total PC Providers** 	|
|---------------------------------------	|:---------------:	|:----------------------------:	|
| CHRISTIANA   CARE HEALTH SERVICES INC 	|       871       	|              226             	|
| _NULL_                                  	|       _513_       	|              _94_              	|
| ANESTHESIA   SERVICES PA              	|       161       	|               0              	|
| BAYHEALTH   MEDICAL CENTER, INC.      	|       155       	|              23              	|
| BEEBE   PHYSICIAN NETWORK INC.        	|       134       	|              48              	|
| ATI HOLDINGS   LLC                    	|       127       	|               0              	|
| THE NEMOURS   FOUNDATION              	|       120       	|              20              	|
| DYNAMIC   THERAPY SERVICES LLC        	|        68       	|               0              	|
| APOGEE MEDICAL   GROUP DELAWARE INC   	|        60       	|              28              	|
| MID SUSSEX   MEDICAL CENTER INC       	|        57       	|              16              	|


3. **With an emphasis placed on** replicating real-world deliverable timelines, I devised a fairly simple way to sort all practices at once – (still able to review and adjust later, if needed). ::Utilizing the indicators in the previous step, I created a threshold requirement of 50% or greater; meaning that if a given practice was comprised of 60% of physicians and providers that met my criteria, that it was safe to classify it as a Primary Care Practice.:: 

``` sql
ALTER TABLE Practice_Pivot_Table ADD [Percent_PCPs] 
AS CAST (CAST(Total_Primary_Care_Providers 
AS FLOAT) / CAST(Total_Providers AS FLOAT) 
AS NUMERIC(12, 2)) * 100 ---creating % column
```
``` sql
ALTER TABLE Practice_Pivot_Table 
  ADD [Threshold_Indicator] AS 
  CAST (
	CASE 
   	WHEN (
		CAST(CAST(Total_Primary_Care_Providers AS FLOAT) / 
		CAST(Total_Providers AS FLOAT) 
		AS NUMERIC(12,2))*100 >= 50.000 --creating threshold check (y/n)
		 )
		THEN 'Yes'
	ELSE 'No'
  END AS VARCHAR(50)
	)
```

This was also spot-checked and briefly validated by picking a handful of practices and manually digging through information like location, address/building size, practice name, provider makeup, etc… ::The threshold criterion was between 90-95% accurate.::. The system identified ::72 practices as Primary Care Practices::. However, since the selection of a 50% threshold cut-off wasn't chosen based on proven, well understood studies regarding PC Practices and their correlated factor relationships, I manually checked a handful of practices with less than 50% of staff being primary care. I discovered an additional 18 practices which were added after the fact by hard-coding their entries. Using this, I arrived at an answer to **question #1**: There are ::**90** Primary Care Practices:: in the state of Delaware, as shown below, according to my methodology:

| **Practice Name**                                  	| **Total Providers**	| **Total PC Providers** 	| **PCP %s** 	|  **Threshold Indicator**  	|
|--------------------------------------------------	|:---------------:	|:----------------------------:	|:------------:	|:---------------------:	|
| A DOUGLAS   CHERVENAK DO PA                      	|        2        	|               2              	|      100     	|          Yes          	|
| BANCROFT   FAMILY CARE, PA                       	|        2        	|               2              	|      100     	|          Yes          	|
| BANCROFT   INTERNAL MEDICINE PA                  	|        2        	|               2              	|      100     	|          Yes          	|
| BRANDYWINE   HUNDRED FAMILY MEDICINE LLC         	|        2        	|               2              	|      100     	|          Yes          	|
| COAST TO COAST   HOSPITAL SURGEONS INC           	|        1        	|               1              	|      100     	|          Yes          	|
| COMPLETE   FAMILY CARE, INC                      	|        2        	|               2              	|      100     	|          Yes          	|
| DAVID S ESTOCK   MD PA                           	|        2        	|               2              	|      100     	|          Yes          	|
| DELAWARE   FAMILY CARE ASSOCIATES                	|        2        	|               2              	|      100     	|          Yes          	|
| DELAWARE   MEDICAL GROUP, LLC                    	|        4        	|               4              	|      100     	|          Yes          	|
| DOVER FAMILY   PHYSICIANS PA                     	|        6        	|               6              	|      100     	|          Yes          	|
| FAMILY HEALTH   OF DELAWARE                      	|        2        	|               2              	|      100     	|          Yes          	|
| FAMILY MEDICAL   ASSOCIATES OF DELAWARE          	|        2        	|               2              	|      100     	|          Yes          	|
| FAMILY MEDICAL   CENTRE                          	|        2        	|               2              	|      100     	|          Yes          	|
| FAMILY   MEDICINE OF MIDDLETOWN, PA.             	|        2        	|               2              	|      100     	|          Yes          	|
| FIRST STATE   MEDICAL ASSOCIATES                 	|        3        	|               3              	|      100     	|          Yes          	|
| GEORGETOWN   FAMILY MEDICINE                     	|        2        	|               2              	|      100     	|          Yes          	|
| HOCKESSIN   FAMILY MEDICINE LLC                  	|        2        	|               2              	|      100     	|          Yes          	|
| HORIZONS   FAMILY PRACTICE PA                    	|        2        	|               2              	|      100     	|          Yes          	|
| INTERNAL   MEDICINE ASSOCIATES PA                	|        3        	|               3              	|      100     	|          Yes          	|
| INTERNAL   MEDICINE OF DOVER PA                  	|        5        	|               5              	|      100     	|          Yes          	|
| KATHRYN L FORD   FAMILY PRACTICE CENTER, L.L.C.  	|        2        	|               2              	|      100     	|          Yes          	|
| LMG   PERIOPERATIVE MEDICAL CONSULTANTS, PA, INC 	|        3        	|               3              	|      100     	|          Yes          	|
| LOUGHRAN   MEDICAL GROUP, P.A.                   	|        4        	|               4              	|      100     	|          Yes          	|
| MEDICAL   ASSOCIATES OF BEAR                     	|        2        	|               2              	|      100     	|          Yes          	|
| MIDDLETOWN   FAMILYCARE ASSOCIATES LLC           	|        2        	|               2              	|      100     	|          Yes          	|
| MILFORD   MEDICAL ASSOCIATES, PA                 	|        5        	|               5              	|      100     	|          Yes          	|
| QUALITY FAMILY   PHYSICIANS, PA                  	|        3        	|               3              	|      100     	|          Yes          	|
| RELIANCE   HEALTHCARE, LLC                       	|        2        	|               2              	|      100     	|          Yes          	|
| SETH L IVINS   MD LLC                            	|        3        	|               3              	|      100     	|          Yes          	|
| SOUTHSIDE   FAMILY PRACTICE, P.A.                	|        2        	|               2              	|      100     	|          Yes          	|
| THE FAMILY   DOCTORS PA                          	|        2        	|               2              	|      100     	|          Yes          	|
| THE FAMILY   PRACTICE CENTER OF NEW CASTLE, PA   	|        2        	|               2              	|      100     	|          Yes          	|
| TRINITY   MEDICAL ASSOCIATES, PA                 	|        2        	|               2              	|      100     	|          Yes          	|
| WILLIAM B.   FUNK, M.D., PA                      	|        2        	|               2              	|      100     	|          Yes          	|
| WILMINGTON   MEDICAL ASSOCIATES, P.A.            	|        3        	|               3              	|      100     	|          Yes          	|
| SOUTHBRIDGE   MEDICAL ADVISORY COUNCIL INC       	|        6        	|               5              	|      83      	|          Yes          	|
| DELAWARE   MEDICAL ASSOCIATES PA                 	|        5        	|               4              	|      80      	|          Yes          	|
| DELAWARE   POST-ACUTE MEDICAL SERVICES 1 PA      	|        5        	|               4              	|      80      	|          Yes          	|
| MILTON   ENTERPRISES INC.                        	|        5        	|               4              	|      80      	|          Yes          	|
| DELAWARE   PRIMARY CARE LLC                      	|        4        	|               3              	|      75      	|          Yes          	|
| TOTAL CARE   PHYSICIANS, P.A.                    	|        11       	|               8              	|      73      	|          Yes          	|
| HOSPITAL   MEDICINE ASSOCIATES LLC               	|        31       	|              22              	|      71      	|          Yes          	|
| SEAFORD   INTERNAL MEDICINE LLC                  	|        7        	|               5              	|      71      	|          Yes          	|
| STONEY BATTER   FAMILY MEDICINE ASSOCIATES PA    	|        10       	|               7              	|      70      	|          Yes          	|
| INPATIENT   CONSULTANTS OF DELAWARE, INC         	|        35       	|              24              	|      69      	|          Yes          	|
| BRANDYWINE   MEDICAL ASSOCIATES, INC.            	|        3        	|               2              	|      67      	|          Yes          	|
| CENTRAL   DELAWARE FAMILY MEDICINE               	|        3        	|               2              	|      67      	|          Yes          	|
| FAMILY CARE   ASSOCIATES, INC                    	|        3        	|               2              	|      67      	|          Yes          	|
| NORTH BAY   MEDICAL ASSOCIATES PA                	|        3        	|               2              	|      67      	|          Yes          	|
| POST ACUTE   PHYSICIAN PARTNERS LLC              	|        3        	|               2              	|      67      	|          Yes          	|
| SOUTHERN   DELAWARE MEDICAL GROUP                	|        3        	|               2              	|      67      	|          Yes          	|
| UNITED MEDICAL   CLINIC OF DE LLC                	|        6        	|               4              	|      67      	|          Yes          	|
| VAN BUREN   MEDICAL ASSOCIATES                   	|        6        	|               4              	|      67      	|          Yes          	|
| MID-ATLANTIC   FAMILY PRACTICE                   	|        12       	|               7              	|      58      	|          Yes          	|
| FAMILY   MEDICINE AT GREENHILL                   	|        7        	|               4              	|      57      	|          Yes          	|
| AMNA MEDICAL   CENTER, LLC                       	|        11       	|               6              	|      55      	|          Yes          	|
| BETHANY   PRIMARY CARE                           	|        2        	|               1              	|      50      	|          Yes          	|
| CAMDEN PRIMARY   CARE L.L.C                      	|        2        	|               1              	|      50      	|          Yes          	|
| CAMDEN WALK-IN   LLC                             	|        2        	|               1              	|      50      	|          Yes          	|
| EINSTEIN   PRACTICE PLAN INC                     	|        2        	|               1              	|      50      	|          Yes          	|
| FAMILY   PRACTICE OF HOCKESSIN, P.A.             	|        2        	|               1              	|      50      	|          Yes          	|
| FIRST STATE   FAMILY PRACTICE, INC               	|        2        	|               1              	|      50      	|          Yes          	|
| FRIENDS AND   FAMILY PRACTICE, PA                	|        2        	|               1              	|      50      	|          Yes          	|
| GERIATRIC   MEDICINE CONSULTANTS PA              	|        2        	|               1              	|      50      	|          Yes          	|
| LA RED HEALTH   CENTER INC                       	|        2        	|               1              	|      50      	|          Yes          	|
| LAUREL MEDICAL   GROUP, LLC                      	|        2        	|               1              	|      50      	|          Yes          	|
| MARIE C   WOLFGANG MD                            	|        2        	|               1              	|      50      	|          Yes          	|
| MB MEDICAL   SERVICES LLC                        	|        2        	|               1              	|      50      	|          Yes          	|
| NEWARK   EMERGENCY PARTNERS LLC                  	|        2        	|               1              	|      50      	|          Yes          	|
| SUSSEX   INTERNAL MEDICINE LLC                   	|        2        	|               1              	|      50      	|          Yes          	|
| UNITED MEDICAL   LLC                             	|        2        	|               1              	|      50      	|          Yes          	|
| ZARRAGA AND   ZARRAGA INTERNAL MEDICINE PA       	|        2        	|               1              	|      50      	|          Yes          	|
| APOGEE MEDICAL   GROUP DELAWARE INC              	|        60       	|              28              	|      47      	| Yes - Manual Override 	|
| FAST CARE   MEDICAL AID UNIT LLC                 	|        9        	|               4              	|      44      	| Yes - Manual Override 	|
| MEDEXPRESS   INC-DELAWARE                        	|        39       	|              16              	|      41      	| Yes - Manual Override 	|
| ZAREK DONOHUE   LLC                              	|        8        	|               3              	|      38      	| Yes - Manual Override 	|
| WESTSIDE   FAMILY HEALTHCARE, INC.               	|        33       	|              12              	|      36      	| Yes - Manual Override 	|
| BEEBE   PHYSICIAN NETWORK INC.                   	|       134       	|              48              	|      36      	| Yes - Manual Override 	|
| HERITAGE   MEDICAL ASSOCIATES, P.A.              	|        3        	|               1              	|      33      	| Yes - Manual Override 	|
| PREMIERE   PHYSICIANS P.A.                       	|        3        	|               1              	|      33      	| Yes - Manual Override 	|
| ATLANTIC ADULT   AND PEDIATRIC MEDICINE PA       	|        3        	|               1              	|      33      	| Yes - Manual Override 	|
| THE PEARL   CLINIC, LLC                          	|        3        	|               1              	|      33      	| Yes - Manual Override 	|
| CHRISTIANA   CARE HEALTH INITIATIVES             	|        10       	|               3              	|      30      	| Yes - Manual Override 	|
| MID SUSSEX   MEDICAL CENTER INC                  	|        57       	|              16              	|      28      	| Yes - Manual Override 	|
| EDEN HILL   EXPRESS CARE LLC                     	|        16       	|               4              	|      25      	| Yes - Manual Override 	|
| PENINSULA   REGIONAL MEDICAL CENTER              	|        12       	|               3              	|      25      	| Yes - Manual Override 	|
| REGIONAL   MEDICAL GROUP LLC                     	|        4        	|               1              	|      25      	| Yes - Manual Override 	|
| ATLANTIC   GENERAL HOSPITAL CORP                 	|        9        	|               2              	|      22      	| Yes - Manual Override 	|
| ALPHA CARE   MEDICAL LLC                         	|        5        	|               1              	|      20      	| Yes - Manual Override 	|
| PREMIER   IMMEDIATE MEDICAL CARE DELAWARE, LLC   	|        7        	|               1              	|      14      	| Yes - Manual Override 	|



4. **The next goal** is attributing the total volume of Medicare Annual Wellness Visits in 2016. First, since these totals are based on the entire practice's contribution, we can widen our capture range by including visits given by non-primary care providers. Our first goal is to create a master list of all providers that belong to a primary care practice. I created a table by using our ___Practice Pivot Table___ as a foundation, while left-joining our ___physician compare table___ on the practice name:

``` sql
SELECT a.Practice_Name
	  ,b.npi
INTO Testing_Table ---creating new table
FROM Practice_Pivot_Table AS a
LEFT JOIN [dbo].[physician_compare_table] AS b ON a.Practice_Name = b.organization_legal_name
WHERE a.Threshold_Indicator = 'Yes'
```
After removing duplicates, we now have __715 providers__ across __90 primary care practices__ in the state of Delaware.

5. **At this point**, all that's left to complete question #2 is linking Primary Care Practices and their respective individual staff members to the Annual Wellness Visit data found in the ___physician supplier hcpcs table___. The query below does the heavy lifting of joining, aggregating, and organizing the results for us, which can be found in the table beneath the code block; showing the top 10 Primary Care Practices with regards to their total volume of Annual Wellness Visits. This answers **question #2**; as the ::Primary Care Practice that performed the highest volume of Medicare Annual Wellness Visits in Delaware in 2016, was **Beebe Physician Network Inc**.:: - with an approximated 2,344 total visits through their providers.

``` sql
SELECT b.Practice_Name
	,SUM(a.line_srvc_cnt) AS Total_Wellness_Visits
FROM [dbo].[physician_supplier_hcpcs_table] AS a
LEFT JOIN Testing_Table AS b ON b.npi = a.npi
WHERE (
		a.hcpcs_code = 'G0439'
		OR a.hcpcs_code = 'G0438'
		)
	AND b.Practice_Name IS NOT NULL
GROUP BY b.Practice_Name
ORDER BY SUM(a.line_srvc_cnt) DESC
```

| **Primary Care Practice**                             	| **Total Wellness Visits** 	|
|---------------------------------------------	|:---------------------:	|
| BEEBE PHYSICIAN NETWORK INC.                	|          2,344         	|
| MID-ATLANTIC FAMILY PRACTICE                	|          2,108         	|
| MILTON ENTERPRISES INC.                     	|          1,320         	|
| STONEY BATTER FAMILY MEDICINE ASSOCIATES PA 	|          1,285         	|
| MILFORD MEDICAL ASSOCIATES, PA              	|          923          	|
| ATLANTIC ADULT AND PEDIATRIC MEDICINE PA    	|          829          	|
| SOUTHERN DELAWARE MEDICAL GROUP             	|          824          	|
| SEAFORD INTERNAL MEDICINE LLC               	|          756          	|
| DELAWARE PRIMARY CARE LLC                   	|          740          	|
| APOGEE MEDICAL GROUP DELAWARE INC           	|          739          	|

6. To be continued... almost done.
