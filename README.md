# Project Showcase: End-to-End Spend Analytics Solution
Present is a real customer case, but respecting confidentiality all data was replaced with dummy and any database schemas are high-level approximations.

## Motivation
Customer's procurement team was underserved from Technology / Analytics perspective and suffered from the limited access to spend data. This directly impacted their efficiency and ability to find cost optimizaton opportunities, identify potential compliance risks or track spend trends overtime. 

The only tool they had was macro-enabled Excel document (further refered as 'master'), which was only capable of providing high-level monthly summary metrics per supplier, was not user friendly to a general business user, and was tedious to maintain. Team had access to all the raw data and tools, but lacked the skillset to convert them into fully operational spend analytics solution.

## Legacy solution
Legacy tool was hard to maintain, could not be used by an average business user, did not scale well and couldn't provide required level of details. 

High-level Legacy workflow:
1. Analyst would schedule multiple queries in PeopleSoft to pull 4 tables monthly and manually download files.
2. Manually adjust naming conventions in the newly generated files, clean reference data, compare existing reference files to new and implement changes to existing files linked to the master.
3. Gather additional data from peers regarding updates to reference data (like list of active Business Units).
4. Refresh querries in the master.
5. Once a year update master model to include all 12 months data

## Toolkit
In the interest of maintainability technology choice was dictated by the existing tools used by the customer:
- Peoplesoft - ERP;
- Alteryx - ETL engine and orchestration;
- FSx File Server - storage where Alteryx server has read/write access to flat-files;
- Tableau - BI Platform.

## Proposed Solution Overview
 Automated ETL(Extract-Transform-Load) Pipeline from ERP to BI Dashboard, batch-processing AP data, cleansing and enriching with data from other enterprise systems. High-level pipeline overview:
 - Alteryx Server triggers workflow on monthly schedule;
 - Workflow pulls PeopleSoft tables, combines them, cleans, merges with historical data and then enriches with data from files on S3, and FSx Fileshare; 
 - Processed dataset is pushed to Tableau server and saved on FSx as Alteryx Database file.
 
![image](assets/kitchen_sink-data_pipeline.jpeg) 


## Results
New Spend Analytics Solution had multiple effects on the team:
- For the first time team gained access to their spend data at scale and was able to compare the spend by category, supplier, time period or geography;
- Such visibility enabled team to identify over $120M worth of cost-optimization opportunities and deploy 2-year program to execute them;
- Business users can now self-serve analytics without need to rely on more tech-savvy teammates, procurement data is now widely available across all orgs;
- Time required to maintain the spend analytics solution has  decreased from 120 to just 2 hours per year;
- This foundational pipeline allowed team to work on a deeper layers of analysis combining spend data with Sales, Logistics and Contract data.

## Detailed Process
If you are not interested in the technical details of how the solution was built you can stop reading now. 

### Extract
So the first step was to get the data from the ERP, but before we dive into detailes of the table schema, there are few things you need to know about Accounta Payable (AP) lifecycle and records. 

You are probably familiar with the general flow of **Purchase Order created** -> **Goods delivered / Services Rendered** -> **Invoice received** -> **Payment**. But Accounting is not there only to support actual transaction,they also take care of bookkeeping. So some AP specifics you may not be aware of is that between reciving the invoice and submitting it for payment AP team would normally create additional abstractions called **Voucher** and **Distribution line**. 

**Voucher** is a formal record, usually internal, that authorizes the payment of a vendor invoice. It summarizes the key details of the purchase, including: Vendor information, Invoice date and number, Purchase order number (if applicable), Description of goods or services, Amount owed, Payment terms, Approval signatures. The voucher serves as a central document for tracking the invoice from receipt to payment. It ensures that all necessary information is readily available and verifies that the payment is valid before it's processed.

**A distribution line** is a sub-component of a voucher that specifies how the total payment amount will be allocated across different accounts in the company's general ledger. It typically includes: Account code(s), Department or cost center, Project or other internal reference, Amount to be distributed to each account.

By using distribution lines, companies can track the costs associated with specific purchases and ensure accurate financial reporting. For example, if an invoice covers office supplies for different departments, you might have separate distribution lines for each department's expense account.

This context might help you make little more sense about the below schema. 

```mermaid
---
title: Peoplesoft Table schema
---
erDiagram
    VOUCHER ||--|{ DISTRIBUTION_LINES:"one VOUCHER-BU pair has one or more DISTRIBUTION LINES"
    VOUCHER {
        VOUCHER_ID VARCHAR2(8)  PK
        INVOICE_ID VARCHAR2(30) 
        INVOICE_DATE DATE 
        ACCOUNTING_DATE DATE 
        DUE_DATE DATE 
        VENDOR_ID VARCHAR2(10)  FK
        PAYMENT_TERMS VARCHAR2(5) 
        BUSINESS_UNIT VARCHAR2(5)  PK, FK
        INV_ORIGIN VARCHAR2(30)
        ENTRY_STATUS VARCHAR2(1)
    }
    VENDOR ||--o{ VOUCHER:"one or more VENDOR has 0 or more VOUCHERS"
    VENDOR {
        VENDOR_ID VARCHAR2(10)  PK 
        NAME VARCHAR2(40) 
    }
    DISTRIBUTION_LINES {
        VOUCHER_ID VARCHAR2(8)  PK, FK
        VOUCHER_LINE_ID NUMBER(38)  PK
        LINE_DESCR VARCHAR2(30)
        BUSINESS_UNIT VARCHAR2(5)  FK
        PO_ID VARCHAR2(10) 
        PROJECT_ID VARCHAR2(15) FK
        ACCOUNT_ID VARCHAR2(10) FK
        MONETARY_AMT NUMBER(26)
    }
    ACCOUNT ||--|{ DISTRIBUTION_LINES:"one DISTRIBUTION LINE allocated to one or more ACCOUNTS"
    ACCOUNT {
        ACCOUNT_ID VARCHAR2(10) PK
        ACCOUNT_DESC VARCHAR2(30)
    }
    PROJECT ||--o{ DISTRIBUTION_LINES:"one DISTRIBUTION LINE attributed to 0 or one PROJECTS"
    PROJECT {
        PROJECT_ID VARCHAR2(15) PK
        PROJECT_DESC VARCHAR2(30)
    }

```
 Every record in `VOUCHER` table matches to at least one `DISTRIBUTION LINE`. And reference tables contain additional dimensions for `VENDOR`, `ACCOUNT` and `PROJECT` objects. SQL query seems straigthforward, however `ACCOUNT` and `VENDOR` tables contained duplicate keys, meaning these tables have many-to-many relationship to `VOUCHER` and `DISTRIBUTION LINE`. As identified later this was because both tables contained records about inactive / historical accounts and vendors. This complexity still could be overcome with use of window functions and common table expressions or temporary tables, so the query could look like this:

``` sql
WITH VENDOR AS (
    SELECT VENDOR_ID, NAME1
    FROM(
        SELECT VENDOR_ID,
            NAME1,
            VENDOR_STATUS,
            ROW_NUMBER() OVER(PARTITION BY VENDOR_ID ORDER BY VENDOR_STATUS) RN
        FROM PS_VENDOR)
    WHERE RN = 1),

    ACC AS (
    SELECT ACCOUNT,
	    DESCR
    FROM(
        SELECT  ACCOUNT, 
            DESCR,
            EFFDT,
            EFF_STATUS,
			ROW_NUMBER() OVER(PARTITION BY ACCOUNT ORDER BY EFFDT DESC, EFF_STATUS) RN
        FROM PS_GL_ACCOUNT_TBL 
        )
    WHERE RN=1
    )

SELECT A.VOUCHER_ID,
    A.INVOICE_ID,
    A.VENDOR_ID,
    C.NAME1 AS VENDOR_NAME,
    A.INVOICE_DT,
    A.ACCOUNTING_DT,
    A.OPRID AS INV_ORIGIN,
    A.VCHR_TTL_LINES AS INV_LINE_COUNT,
    A.GROSS_AMT AS INV_TOTAL,
    B.MONETARY_AMOUNT,
    B.ACCOUNT,
    ACC.DESCR AS ACCOUNT_DESCR,
    B.PROJECT_ID,
    D.DESCR AS PROJECT_DESCR
        
FROM PS_VOUCHER A
    JOIN PS_DISTRIB_LINE B ON A.BUSINESS_UNIT = B.BUSINESS_UNIT
        AND A.VOUCHER_ID = B.VOUCHER_ID
    JOIN VENDOR C on A.VENDOR_ID = C.VENDOR_ID 
    JOIN ACC on B.ACCOUNT = ACC.ACCOUNT
    LEFT JOIN PS_PROJECT D on B.PROJECT_ID = D.PROJECT_ID
    
WHERE A.ACCOUNTING_DT >= TO_DATE('2023-02-01','YYYY-MM-DD') 
    and A.ACCOUNTING_DT <= TO_DATE('2023-02-02','YYYY-MM-DD') --Dates are used as placeholders and overriden with Alteryx Dynamic Input tool
    
ORDER BY A.BUSINESS_UNIT, A.VOUCHER_ID, B.VOUCHER_LINE_NUM

```