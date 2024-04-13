# AMH2 - EFT Insurance Payment Posting

Status: Design
Delivery Manager: Jodi Silverman
Design owner: Jodi Silverman

### Document History

| Date | Details | Status | Author |
| --- | --- | --- | --- |
| 01/25/23 | Started MAP creation |  |  |
| 03/12/23 | Review MAP with Customer |  |  |

<aside>
<img src="https://www.notion.so/icons/folder_lightgray.svg" alt="https://www.notion.so/icons/folder_lightgray.svg" width="40px" /> **Table of Contents**

</aside>

# I. Introduction

### **Summary**

The Payment Posting process entails the bot accessing Cadence Bank and Waystar portals to retrieve newly available EFTs. Upon analyzing the transactions, amounts, and other information contained in each EFT, the bot will then proceed to update the records in Traumasoft and post the batches accordingly. Subsequently, it will generate a comprehensive report documenting the performed actions and any noteworthy exceptions.

### Definitions

`EOB`  - Explanation of Benefits - Shows the patient the total charges for their visit. It is not a bill but rather assists in understanding the extent of coverage provided by their health plan and the amount they will be responsible for paying upon receiving a bill from their healthcare provider.

`EFT` - Electronic Funds Transfer - A process to transfer funds electronically from one account to another.

`Crossover Payor` - A crossover payor refers to a secondary insurance provider in situations where an individual has more than one type of insurance coverage. This process, known as coordination of benefits, involves the primary and secondary insurance companies interacting with each other to determine the division of cost for covered services.

### Other Important Links

[Walkthrough Videos](https://drive.google.com/drive/folders/1Ox-GMw-sZ7bn1ZJqvyhc1CJgKWaAhvI-?usp=drive_link)

# II. Workflow Overview

***IMPORTANT DEVELOPMENT NOTE:** There is no sandbox and almost everything you do can be deleted.  During development we can test importing batches and working with batches, however we must delete them when we are done.  Also, for specific rules and functions we will need to work with batches that have not yet been imported and not yet closed.  We will need to get these from AMH when we are ready.*

<aside>
💡 **TRAUMASOFT SCREENS**

- **Electronic Remittance Processing Form**
    
    ![Untitled](AMH2%20-%20EFT%20Insurance%20Payment%20Posting%20ddea2c7b736248b999644d69a526426f/Untitled.png)
    
- **Credit Processing Form**
    
    ![Untitled](AMH2%20-%20EFT%20Insurance%20Payment%20Posting%20ddea2c7b736248b999644d69a526426f/Untitled%201.png)
    
- **Batch Entry Table**
    
    ![Untitled](AMH2%20-%20EFT%20Insurance%20Payment%20Posting%20ddea2c7b736248b999644d69a526426f/Untitled%202.png)
    
- **Add/Edit Credit Form**
    
    ![image.png](AMH2%20-%20EFT%20Insurance%20Payment%20Posting%20ddea2c7b736248b999644d69a526426f/image.png)
    
</aside>

## Functions

### Download EOBs and 835s function

*Function Arguments: `fn_eft`*

1. Get the Empower Value
    1. Navigate to the `WS Payers` tab of the `AMH2 Mapping.xlsx` file
    2. Filter the `WS ID` column by `{eft.waystar_id}`
    3. Update `{eft.empower}` = the corresponding `EMPOWER` value
    4. Update `{eft.payor}` = the corresponding `TRAUMASOFT PAYOR` value
2. Download the 835 File
    1. Click on the `V` (down arrow) in the `Action` column
    2. Select the `Download` option
    3. Rename the file to “DEP `{eft.txn_date}`_`{eft.waystar_id}`_`{eft.amount}`.835.EDI”
        
        *Note:*  `*{eft.txn_date}` should be in yyyymmdd format*
        
3. Download the EOB
    1. Click on the `EOB` icon
    2. Click on the `Download` icon in the `EOB` window
    3. Save the file as “DEP `{eft.txn_date}`_`{eft.waystar_id}`_`{eft.amount}`.pdf”
        
        *Note:*  `*{eft.txn_date}` should be in yyyymmdd format*
        
    4. Close the `EOB` window
4. Update EFT Record
    1. Update `{eft.matched}` = “Y”
    2. Update `{eft.835_file}` = “DEP `{eft.txn_date}`_`{eft.waystar_id}`_`{eft.amount}`.835.EDI”
        
        *Note:*  `*{eft.txn_date}` should be in yyyymmdd format*
        

### Add New Payor to Patient Record Function

*Function Arguments: `fn_txn`, `fn_eft`, `fn_amt`*

1. Navigate to the Call Properties
    1. Open a new browser tab
    2. Go to “https://allmh.traumasoft.com/main.php?a=billing:validation/main&popup=1#id:`fn_txn["leg_id"]`”
    3. Click on the `Edit` button in the `Customer` section
2. Add Payor
    1. Navigate to the `Payors` section in the new browser tab
    2. Click on the `[New Payor]` hyperlink
    3. Type `fn_eft["payor"]` in the `Payor` input of the `Add Payor` window
    4. Double-click the matching `Payor Name` in the `Select Payor` window
    5. Type “1” in the `Order` input **IF** `fn_txn["status"]` = “Primary” **ELSE** 
    Type “2” in the `Order` input **IF** `fn_txn["status"]` = “Secondary” **ELSE** Type “3”
    6. Click on the `OK` button
    7. Click on the `OK` button in the `Traumasoft System Message` window
3. Check Eligibility
    1. Click on the row in the `Payors` table **WHERE** `fn_eft["payor"]` = `Payor Name`
    2. Get Eligibility Information
        1. Click on the `Check Eligibility` hyperlink in the `actions` column
        2. Select `Manual` from the options in the `Select Eligibility Service` window
        3. Type `fn_eft["dos"]` in the `Date:` input
        4. Click on the `OK` button
        5. Click on the `Check Eligibility` hyperlink again
        6. Select `Payor Logic` from the options in the `Select Eligibility Service` window
        7. Click on the `OK` button
    3. Is `Status:` = “Active Coverage” in the `View Results` window?
        1. Y - Take the following actions:
            - Get the `subscriber_id` = `Subscriber ID:`
            - Click on the `Close` button
            - Skip to the [Add Subscriber ID](https://www.notion.so/Add-Subscriber-ID-97c0ebc5ee314c67b24d1560eebcfe05?pvs=21) step
        2. N - Go to the next step
    4. Run the [Add Transaction Change Function](https://www.notion.so/Add-Transaction-Change-Function-591970dcba214ec6a3f5eaa00f70326e?pvs=21) 
        1. `fn_txn_id` = `fn_eft["id"]`
        2. `fn_type` = “Exception”
        3. `fn_amt` = `fn_amt`
        4. `fn_note` = “Payor is Not Showing Active in Payor Logic.”
4. Add Subscriber ID
    1. Click on the `Edit` hyperlink in the `actions` column
    2. Type `subscriber_id` in the `Identification:` input
    3. Click on the `OK` button
    4. Click on the `OK` button in the `Traumasoft System Message` window
5. Close the `Edit Patient` window
6. Close the browser tab

### Add Transaction Function

*Function Arguments: `fn_eft`, `fn_txn_id`*

1. Is there any `fn_txn_id`  **FOR** `{txn}` **IN** `{eft.txns}`?
*Note: Checks if we already have the Transaction ID in our list*
    1. Y - End Function
    2. N - Go to the next step
2. Navigate to the Billing Workflow
    1. Click on the arrow next to the `Billing` button
        
        ![Untitled](AMH2%20-%20EFT%20Insurance%20Payment%20Posting%20ddea2c7b736248b999644d69a526426f/Untitled%203.png)
        
    2. Select `Workflow` from the `Billing` options
3. Select Search Options
    1. Select `Transaction ID` from the `Select One` options
    2. Type `fn_txn_id` into the `Transaction ID` input
    3. Click on the `Search` button
4. Is there more than one row?
    1. Y - Take the following Actions:
        1. Update EFT Record
            - `fn_eft["posted"]` = “P”
            - `fn_eft["notes"]` = “Partially Posted.  Multiple rows Transaction ID search.  ID: `fn_txn["id"]`”
        2. End Function
    2. N - Go to the next step
5. Get the Billing Date of Service
    1. `dos` = `row.Billing DOS`
6. Get the Leg ID
    1. Double-click on the row
    2. Get the `url` from the `Call Properties: Allegiance` window
    3. `leg_id` = numbers after “#id:” at the end of the `url`
        
        ![Untitled](AMH2%20-%20EFT%20Insurance%20Payment%20Posting%20ddea2c7b736248b999644d69a526426f/Untitled%204.png)
        
    4. Close the `Call Properties: Allegiance` window
7. Open the EOB pdf
    1. `eob_url` = “https://allmh.traumasoft.com/api/Billing/BatchEobFile?rtype=GetByLegId&legId=`leg_id`”
    2. Navigate to a new browser tab
    3. Go to `eob_url`
    4. Save EOB as “`fn_txn_id`_EOB.pdf”
8. Get the Claim Status    *(See EOB Examples Below)*
    1. Search for the first instance of “Processed as ”
    2. `claim_status` = the text following “Processed as ” **AND** before the “,”
9. Get the Crossover Payor    *(See EOB Examples Below)*
    1. Does the text also contain “Additional Payer(s): “?
        1. Y - `crossover` = the text following “Additional Payer(s): ”
        2. N - `crossover` = “”
10. Create Transaction Record
    1. `{txn.id}` = `fn_txn_id`
    2. `{tnx.leg_id}` = `leg_id`
    3. `{tnx.status}` = `claim_status`
    4. `{txn.crossover}` = `crossover`
    5. `{txn.dos}` = `dos`
11. Append `{txn}` record to `fn_eft["txns"]` list
12. Close the browser tab

<aside>
💡 **EOB Examples**

![Untitled](AMH2%20-%20EFT%20Insurance%20Payment%20Posting%20ddea2c7b736248b999644d69a526426f/Untitled%205.png)

</aside>

### Add Transaction Change Function

*Function Arguments: `fn_txn_id`, `fn_type`, `fn_amt`, `fn_note`*

1. Create New Change Record
    1. `{change.type}` = `fn_type`
    2. `{change.amount}` = `fn_amt`
    3. `{change.note}` = `fn_note`
2. Find the ****`fn_txn_id` **FOR** `{txn}` **IN** `{eft.txns}`
3. Add `{change}` record to `{txn.changes}`

### Zero Out Forward Balance Function

*Function Arguments: `fn_eft`*

1. Navigate to the `Credit Processing` section
2. Click on the `Edit` button in the `Credit` row
3. Update `Amount` = 0.00
4. Navigate to the `Trip` section
5. Does `Current Schedule:` = “Pending Recoupment”?
    1. Y - Take the following actions:
        1. Select the `blank` option from the `Next Payor` options
        2. Select the `blank` option from the `Next Schedule` options
        3. Select the `blank` option from the `Next Event` options
            
            ![Untitled](AMH2%20-%20EFT%20Insurance%20Payment%20Posting%20ddea2c7b736248b999644d69a526426f/Untitled%206.png)
            
    2. N - Take the following actions:
        1.  Select `fn_eft["payor"]` from the `Next Payor` options
        2. Select `Denied Claim` from the `Next Schedule` options
        3. Select `ON HOLD` from the `Next Event` options
6. Type “Zeroed Out Forwarding Balance” in  `Comments:` input
7. Click on the `OK` button

### Add Credit Function

*Function Arguments: `fn_txn_id`*

1. Click on the `fn_eft["payor"]` in the `Batch Entries` table
2. Navigate to the `Credit Processing` form
3. Enter `fn_txn.id` in the `Transaction Id:` input
4. Click on the `Add` button in the `Credit` row

### Add Take Back Function

*Function Arguments: `fn_eft`, `fn_txn_id`, `fn_amt`, `fn_applies_to`, `fn_comment`*

1. Run the [Add Credit Function](https://www.notion.so/Add-Credit-Function-3e15155eebfb4412996ce728b1e02e98?pvs=21) *(See Add Edit Credit Form)*
    1. `fn_txn_id` = `fn_txn_id`
2. Select `Payment - Check` from the `Credit:` options
3. Select `fn_eft["payor"]` from the `Payor:` options
4. Type `fn_amt` in the `Amount` input
5. Is `fn_applies_to` = “”?
    1. Y - Skip to the [Complete the Entry](https://www.notion.so/Complete-the-Entry-79cee51235a44e3aa225eeb9729d73c7?pvs=21) step
    2. N - Go to the next step
6. Check the `Credit applies to ...` checkbox
7. Does `fn_applies_to` = “mileage”?
    1. Y - Select the option that **DOES** contain “mileage” from the `Credit applies to ...` options
    2. N - Select the first option that does **NOT** contain “mileage” from the `Credit applies to ...` options
8. Complete the Entry
    1. Type `fn_comment` in the `Comments:` input
    2. Click on the `OK` button

### Add Write Off Function

*Function Arguments: `fn_eft`, `fn_txn_id`, `fn_amt`, `fn_applies_to`, `fn_comment`*

1. Run the [Add Credit Function](https://www.notion.so/Add-Credit-Function-3e15155eebfb4412996ce728b1e02e98?pvs=21) 
    1. `fn_txn_id` = `fn_txn_id`
2. Select `Manual Contractual Allowance` from the `Credit:` options
3. Select the `fn_eft["payor"]` from the `Payor:` options
4. Type `fn_amt` in the `Amount` input
5. Is `fn_applies_to` = “”?
    1. Y - Skip to the [Complete the Entry](https://www.notion.so/Complete-the-Entry-2690874aac43434690ed6e4d1201c920?pvs=21) step
    2. N - Go to the next step
6. Check the `Credit applies to ...` checkbox
7. Select `fn_applies_to` from the `Credit applies to ...` options
8. Complete the Entry
    1. Type `fn_comment` in the `Comments:` input
    2. Click on the `OK` button

### Change Payment to Denial Function

*Function Arguments: `fn_eft`, `fn_txn_id`, `fn_current_payor`, `fn_comments`*

1. Click on the `fn_eft["payor"]` in the `Batch Entries` table
2. Find the first row in the `Credits` table **WHERE** `Transaction Id` = `fn_txn_id` **AND** `Credit` = “Payment - Check”
3. Click on the `Edit` button in the `Credit` row 
4. Navigate to the `Trip` section
5.  Select `fn_current_payor` from the `Next Payor` options
6. Select `Denied Claim` from the `Next Schedule` options
7. Select `ON HOLD` from the `Next Event` options
8. Check the `Apply next payor, schedule...` checkbox
9. Type `fn_comments` in the `Comments:` input

### Add Interest Function

*Function Arguments: `fn_eft`, `fn_txn_id`, `fn_interest`, `fn_comment`*

1. Open the Add Payment Window
    1. Navigate to the `Credit Processing` form
    2. Type `fn_txn_id` in the `Transaction Id:` input
    3. Click on the `Add by Charge` button in the `Credit:` section
2. Enter the Payment Information
    1. Select the first row where `Charge` does **NOT** contain “Mileage” **OR** “Disposables”
    2. Select `fn_eft["payor"]` from the `Payor` options
    3. Type `fn_interest` in the `Paid` input
    4. Select `Late Fee/Interest Payment` from the `Paid` options
    5. Select -`fn_interest` in the `Adjustment` input
    6. Select `Manual Contractual Allowance` from the `Adjustment` options
3. Type `fn_comment` in the `Comments:` input
4. Click on the `OK` button

### Update Next Payor Function

*Function Arguments:  `fn_crossover`, `fn_comments`*

1. Click on the `Edit` button in the `Credit` row of the `Credit Processing` form
2. Update Crossover Payor
    1. Select `fn_crossover` from the `Next Payor:` options
    2. Select `Automatic Crossover` from the `Next Schedule:` options
    3. Select `AUTOMATIC CROSSOVER` from the `Next Event:` option
3. Type `fn_comments` in the `Comments:` input
4. Click on the `Ok` button

### Add EOB to Patient Account Function

*Function Arguments: `fn_eft`, `fn_leg_id`*

1. Open a new browser tab
2. Navigate to the Call Properties 
    1. Go to “https://allmh.traumasoft.com/main.php?a=billing:validation/main&popup=1#id:`fn_leg_id`"
3. Click on the `Attachments` tab
4. Click on the `Enabling Editing` button
5. Click on the `Add` button in the `Trip Leg Attachments` section
6. Upload the EOB
    1. Click on the `+ Add Files` button in the `Upload Attachments` window
    2. Select the file **WHERE** filename = ****“DEP `{eft.txn_date}`_`{eft.waystar_id}`_`{eft.amount}`.pdf”
    3. Click on the `Open` button
    4. Click on the `Start Upload` button
    5. Close the  `Upload Attachments` window
7. Click on the `Save` button  *(Note: bottom right)*
8. Close the `Call Properties` browser tab

## Macro 1.0 - Preparation and Logins

### Macro 1.1 - Get Selections from Empower

<aside>
💡 **EMPOWER OPTIONS**

**Select Payor(s)**

- [ ]  AETNA
- [ ]  BCBS
- [ ]  CIGNA
- [ ]  HUMANA
- [ ]  MEDICAID
- [ ]  MEDICARE
- [ ]  MOLINA
- [ ]  SUPERIOR
- [ ]  UHC
- [ ]  VA
- [ ]  OTHER

**Options**

- [ ]  Post Batches
</aside>

### Macro 1.2 - Login to Applications

1. Get Bitwarden Credentials
    1. AMH Office 365
    2. AMH Microsoft Graph API Access
    3. AMH Cadence Bank
    4. AMH Traumasoft
    5. AMH Waystar

<aside>
💡 **Manifest Step**

*Step: “Get Bitwarden Credentials”*  

*Description: “Checks to see if we are able to get the Bitwarden Credentials.  If not, the process will end at this step.”*

</aside>

### Macro 1.3 - Get Sharepoint Files

1. Login to Sharepoint with [Get Bitwarden Credentials](https://www.notion.so/Get-Bitwarden-Credentials-3b1d40b95e55419388f6a18b9660560e?pvs=21)
2. Get `AMH2 Mapping.xlsx` [HERE](https://allegianceambulance-my.sharepoint.com/:x:/r/personal/thoughtful_allmh_com/_layouts/15/Doc.aspx?sourcedoc=%7BCBCE1B8B-6912-4141-A453-437B3E6737F9%7D&file=AMH2%20Mapping.xlsx&action=default&mobileredirect=true)
3. Navigate to `Bank Deposit Master.xlsx`  [HERE](https://allegianceambulance-my.sharepoint.com/:x:/r/personal/thoughtful_allmh_com/_layouts/15/Doc.aspx?sourcedoc=%7BE7A5AE1D-C947-42E7-B4C7-C359EEC6A7FF%7D&file=Bank%20Deposit%20Master.xlsx&action=default&mobileredirect=true)  
*Note: We will not be downloading this, we will be updating live*

<aside>
💡 **Manifest Step**

*Step: “Get Sharepoint Files”*  

*Description: “Checks to see if we are able to access the Sharepoint Files.  If not, the process will end at this step.”*

</aside>

### Macro 1.4 - Get Waystar Payers

1. Navigate to the `WS Payers` tab of the `AMH2 Mapping.xlsx` file
2. Filter the `EMPOWER` column = Empower selections
3. For each `row` in filtered `rows:`
    1. Append `row.WS ID` to the `{run.ws_payors}` list

## Macro 2.0 - Prepare Bank Report

### Macro 2.1 - Check Bank Deposit Master

1. Navigate to the `Bank Deposit Master.xlsx`  [HERE](https://allegianceambulance-my.sharepoint.com/:x:/r/personal/thoughtful_allmh_com/_layouts/15/Doc.aspx?sourcedoc=%7BE7A5AE1D-C947-42E7-B4C7-C359EEC6A7FF%7D&file=Bank%20Deposit%20Master.xlsx&action=default&mobileredirect=true) 
2. Are there any records where `TXN DATE` = `{run_date}` - 1 **OR** `{run_date}` - 1 is a Saturday?
    1. Y - Skip to the [Macro 3.0 - Match Deposits and Get Payments](https://www.notion.so/Macro-3-0-Match-Deposits-and-Get-Payments-939335ff6bb744799e57d1e17a0cba6e?pvs=21) step
    2. N - Go to the [Macro 2.2 - Login to Cadence Bank](https://www.notion.so/Macro-2-2-Login-to-Cadence-Bank-4f34a0a726174441931eb33c0738acf5?pvs=21) step

### Macro 2.2 - Login to Cadence Bank

1. Navigate to the Cadence Bank Website
[https://portal.cadencebank.com/signon.aspx](https://portal.cadencebank.com/signon.aspx)
2. Log in using credentials from the [Get Bitwarden Credentials](https://www.notion.so/Get-Bitwarden-Credentials-3b1d40b95e55419388f6a18b9660560e?pvs=21) step
    1. Select the `Send me a code` option
    2. Click on the `By email` radio button
    3. Click on the `Send` button
3. Get Code from Microsoft Outlook
    1. Navigate to Outlook [https://outlook.office.com/](https://outlook.office.com/)
    2. Click on the `Cadence Bank` folder
    3. Click on the most recent message
    4. Copy the `code` from the `Authorization Request` in the body of the email
4. Complete the Login
    1. Enter the `code` in the `Code` input
    2. Click on the `Verify` button
    3. Click on the `No` button under the `Choose a device name` input

<aside>
💡 **Manifest Step**

*Step: “Login to Cadence Bank”*  

*Description: “Checks to see if we are able to Login to Cadence Bank.  If not, the process will end at this step.”*

</aside>

### Macro 2.3 - Get Bank Transactions

1. Select the Operating Account
    1. Navigate to the `Dashboard` tab in the `Home` menu 
    2. Click on the `xxx2200 - Operating` hyperlink under the `Deposit Accounts` section 
    *Note: wait for the page loads*
2. Select the Report Options
    1. Click on the `Posting date` input
        1. Select `Equal to`  into `Date is` input
    2. Is `{run_date}` = Monday?
        1. Y - Type `{run_date}` - 3 into `Date range` input
        2. N - Type `{run_date}` - 1 into `Date range` input 
        3. Click on the `Done` button
3.  Complete the Export
    1. Click on the `Download` icon
    2. Select `CSV` from the options
        
        ![Untitled](AMH2%20-%20EFT%20Insurance%20Payment%20Posting%20ddea2c7b736248b999644d69a526426f/Untitled%207.png)
        
    
    *Note: This will automatically save the report as “Register_MMDDYYYY_HHMMSS.csv”
    Example: “Register_01292024_080832.csv”* 
    

### Macro 2.4 - Update Bank Deposit Master

*Note: The Bank Deposit Master will be used by the client at the same time as we are updating it so it needs to be updated live. 
@Jahanzaib Akhter has done this before. Credentials for Graph API are in BW - AMH Microsoft Graph API Access.  They expire 02/09/2026.*

<aside>
🧪 Is `INPUTS.DATA_SOURCE == DataSource.TEST_DATA` ?

1. Y - `data` = Test Data Set `JSON` file
2. N - `data` = .csv file
</aside>

1. Loop through `data` rows
    1. For each `row` in `data`:
        1. Go to the next step
2. Append a row to the `Bank Deposit Master.xlsx` (`BDM`)
    1. `BDM.CHECK NUM` = `row.checknumber`
    2. `BDM.TXN DATE` = `row.transactiondate`
    3. `BDM.AMOUNT` = `row.Amount`
    4. `BDM.DESC1` = `row.description1`
    5. `BDM.DESC2` = `row.description2`
    6. `BDM.DESC3` = `row.description3`
3. Determine If Deposit is in Waystar
    1. Navigate to the `AMH2 Mapping.xlsx` (`AMH`) file
    2. Does `row.description1` begin with any value in the `AMH.BEGINS WITH` column of the `Waystar` tab?
        1. Y - Go to the next step
        2. N - Take the following actions
            - Append `row.description1` to the `{run.missing_desc1}` list
            - Skip to the[Check for Last Row](https://www.notion.so/Check-for-Last-Row-5ef56785c1f04fab8abe7a17b0fc2985?pvs=21) step
    3. Get the corresponding `AMH.835`
4. Update Remaining Rows
    1. Is the corresponding `AMH.835` = “Y”?
        1. Y - Take the following actions:
            - Update the remaining columns
                - `BDM.835` = “Y”
                - `BDM.WAYSTAR ID` = “”
                - `BDM.MATCHED` = “”
                - `BDM.835 FILE` =  “”
                - `BDM.POSTED` =  “”
            - Skip to the[Check for Last Row](https://www.notion.so/Check-for-Last-Row-5ef56785c1f04fab8abe7a17b0fc2985?pvs=21) step
        2. N - Take the following actions:
            - Update the remaining columns
                - `BDM.835` = “N”
                - `BDM.WAYSTAR ID` = “N/A”
                - `BDM.MATCHED` = “N/A”
                - `BDM.835 FILE` =  “N/A”
                - `BDM.POSTED` =  “N/A”
            - Skip to the[Check for Last Row](https://www.notion.so/Check-for-Last-Row-5ef56785c1f04fab8abe7a17b0fc2985?pvs=21) step
5. Check for Last Row
    1. Is this the last `row` in `data`?
        1. Y - Go to the[Macro 3.1 - Get Unmatched Deposits](https://www.notion.so/Macro-3-1-Get-Unmatched-Deposits-3c23539728a14bcdbab6d861e2431fda?pvs=21) step
        2. N - Go back to the [Loop through `data` rows](https://www.notion.so/Loop-through-data-rows-e94aeca3650746c1af1fbce684cb0ff8?pvs=21) step and the next `row` in the loop

## Macro 3.0 - Match Deposits and Get Payments

<aside>
💡 **DEVELOPMENT NOTE**

*If there are any failures during Macro 3 & 4, set `{eft.posted}` = “N”*

</aside>

### Macro 3.1 - Get Unmatched Deposits

1. Filter Bank Deposit Master
    1. Navigate to `Bank Deposit Master.xlsx`
    2. Filter `835` = “Y” **AND** `MATCHED` = “” **AND** `POSTED` = “” 
2. Get Deposits from Bank Deposit Master
    1. Filter `Bank Deposit Master.xlsx` to 
    2. For each `row` in `rows`:
        1. Create an EFT Object
            - `{eft.check_num}` = `row.CHECK NUM`
            - `{eft.txn_date}` = `row.TXN DATE`
            - `{eft.amount}` = `row.AMOUNT`
            - `{eft.desc1}` = `row.DESC1`
            - `{eft.desc2}` = `row.DESC2`
            - `{eft.desc3}` = `row.DESC3`
            - `{eft.835}`  = `row.835`
            - `{eft.waystar_id}`  = `row.WAYSTAR ID`
            - `{eft.matched}` = `row.MATCHED`
            - `{eft.835_file}` = `row.835 FILE`
            - `{eft.posted}` = `row.POSTED`
            - `{eft.notes}` = `row.NOTES`
            - `{eft.empower}` = `row.EMPOWER`
            - `{eft.payor}` = `row.PAYOR`
        2. Append `{eft}` to `{run.efts}`

### Macro 3.2 - Login to Waystar

1. Log In to Waystar
    1. Navigate to the Waystar Portal ⇒ [https://login.zirmed.com/UI/Login](https://login.zirmed.com/UI/Login)
    2. Log in using credentials from the [Get Bitwarden Credentials](https://www.notion.so/Get-Bitwarden-Credentials-3b1d40b95e55419388f6a18b9660560e?pvs=21) step
        1. Click on the `Log in` button
2. Are there any Full Screen Alerts?
    1. Y - Click the `Continue` button
    2. N - Go to the next step
        
        ![Untitled](AMH2%20-%20EFT%20Insurance%20Payment%20Posting%20ddea2c7b736248b999644d69a526426f/Untitled%208.png)
        

<aside>
💡 **Manifest Step**

*Step: “Login to Waystar Portal”*  

*Description: “Checks to see if we are able to Login to Waystar Portal.  If not, the process will end at this step.”*

</aside>

### Macro 3.3 - Loop Through EFTs

1. For each `{eft}` in `{run.efts}`:
    1. Go to the next step

### Macro 3.4 - Search for EFT

*Note: You should already be on the `Remits` option of the `CLAIMS PROCESSING` window when you log in.  These steps below are to make sure you are on the right tab.*

1. Navigate to the Payments Tab
    1. Click on the `CLAIMS PROCESSING` tab
    2. Select `Remits` from the options
    3. Select `Payment` from the `Remits` menu options
        
        ![Untitled](AMH2%20-%20EFT%20Insurance%20Payment%20Posting%20ddea2c7b736248b999644d69a526426f/Untitled%209.png)
        
2. Enter Search Criteria
    1. Navigate to the `Search` section on the left
    2. Type `{eft.amount}` in the `Amount` input
    3. Select `All` from the `View Options` options
    4. Click on the `Search` button
3. Are there any rows?
    1. Y - Go to the next step
    2. N - Take the following actions:
        1. Append to `{eft.notes}` ⇒ “`{run.date}` TBOT: No payment found on Waystar portal”
        2. Skip to the [Macro 3.6 - Check for Last EFT](https://www.notion.so/Macro-3-6-Check-for-Last-EFT-42ed3a29283b44fba682bf51357d725a?pvs=21) step
            
            *Note: `{run.date}` should be in mm/dd/yyyy format*
            
    
    <aside>
    💡 **Manifest Step**
    
    *Step: “Find Matching Payment on Waystar Portal”*  
    
    *Description: “Checks to see if there is a matching payment on Waystar Portal.”*
    
    </aside>
    
4. Choose the Correct Row
    1. Is ****there is only one row?
        1. Y - Choose that row and skip to the [Macro 3.5 - Download EOBs and 835s](https://www.notion.so/Macro-3-5-Download-EOBs-and-835s-a459bfc4bddb48ce9a35834e7118131f?pvs=21) step
        2. N - Go to the next step
    2.  Is there only one row where `Payment Date` = `{eft.txn_date}`?
        1. Y - Choose that row and skip to the [Macro 3.5 - Download EOBs and 835s](https://www.notion.so/Macro-3-5-Download-EOBs-and-835s-a459bfc4bddb48ce9a35834e7118131f?pvs=21) step
        2. N - Go to the next step
    3. Is there only one row where `Payment Date` ≥ `{eft.txn_date}` -7 **AND** `Payment Date` ≤ `{eft.txn_date}` -7?
        1. Y - Choose that row and skip to the [Macro 3.5 - Download EOBs and 835s](https://www.notion.so/Macro-3-5-Download-EOBs-and-835s-a459bfc4bddb48ce9a35834e7118131f?pvs=21) step
        2. N - Take the following actions:
            1. Append to `{eft.notes}` ⇒ “`{run.date}` TBOT: Multiple payments found on Waystar portal”
            2. Skip to the [Macro 3.6 - Check for Last EFT](https://www.notion.so/Macro-3-6-Check-for-Last-EFT-42ed3a29283b44fba682bf51357d725a?pvs=21) step
                
                *Note: `{run.date}` should be in mm/dd/yyyy format*
                
    
    <aside>
    💡 **Manifest Step**
    
    *Step: “Check for Multiple Payments on Waystar Portal”*  
    
    *Description: “Checks to see if there is more than one matching payment on Waystar Portal.”*
    
    </aside>
    

### Macro 3.5 - Download EOBs and 835s

1. Navigate to the Download Window
    1. Click on the `V` (down arrow) in the `Action` column
    2. Select the `Download` option
    3. Click on the `Go To Downloads` hyperlink in the `Downloads` window
2. Get the Waystar ID
    1. From the `Payer Description` get the text inside the last parenthesis
        
        ![Untitled](AMH2%20-%20EFT%20Insurance%20Payment%20Posting%20ddea2c7b736248b999644d69a526426f/Untitled%2010.png)
        
    2. Update `ws_id` = text
    3. Is `ws_id` in the `{run.ws_payors}` list?
        1. Y - Go to the next step
        2. N - Take the following actions:
            - Remove the `{eft}` from `{run.efts}`
            - Skip to the [Macro 3.6 - Check for Last EFT](https://www.notion.so/Macro-3-6-Check-for-Last-EFT-42ed3a29283b44fba682bf51357d725a?pvs=21) step
3. Update `{eft.waystar_id}` = `ws_id`
4. Run the [Download EOBs and 835s function](https://www.notion.so/Download-EOBs-and-835s-function-0043d2c6cf8f44ee9ab296947c3c2285?pvs=21) 
    1. `fn_eft` = `{eft}`

### Macro 3.6 - Check for Last EFT

1. Is this the last `{eft}` in `{run.efts}`?
    1. Y - Skip to the [Macro 4.0 - Get Zero Pay](https://www.notion.so/Macro-4-0-Get-Zero-Pay-159a7e51183843ee907dfb491ea8e708?pvs=21) step
    2. N - Go back to the [Macro 3.3 - Loop Through EFTs](https://www.notion.so/Macro-3-3-Loop-Through-EFTs-07ede4fafe904ac0a3fb1c54055321b6?pvs=21) step and the next `{eft}` in the loop

## Macro 4.0 - Get Zero Pay

### Macro 4.1 - Search for Zero Pay

1. Navigate to the Payments Tab
    1. Click on the `CLAIMS PROCESSING` tab
    2. Select `Remits` from the options
    3. Select `Payment` from the `Remits` menu options
        
        ![Untitled](AMH2%20-%20EFT%20Insurance%20Payment%20Posting%20ddea2c7b736248b999644d69a526426f/Untitled%209.png)
        
2. Enter Search Criteria
    1. Navigate to the `Search` section on the left
    2. Type “0.00” in the `Amount` input
    3. Type `{run.date}` - 1 into the first `Received Date` input
    4. Type `{run.date}` - 1 into the second `Received Date` input
    5. Select `All` from the `View Options` options
    6. Click on the `Search` button
3. Are there any results?
    1. Y - Go to the next step
    2. N - Skip to the [Macro 5.0 - Begin the Posting Process](https://www.notion.so/Macro-5-0-Begin-the-Posting-Process-263dbe3ac28f4ae3a8b8bb7955f49cb9?pvs=21) step

### Macro 4.2 - Loop through Zero pays

1. Loop through each page
    1. for each `page` in `pages`:
        1. Go to the next step
2. Loop through each row
    1. For each `row` in `rows`:
        1. Go to the next step

### Macro 4.3 - Download Zero Pay EOBs and 835s

1. Navigate to the Download Window
    1. Click on the `V` (down arrow) in the `Action` column
    2. Select the `Download` option
    3. Click on the `Go To Downloads` hyperlink in the `Downloads` window
2. Get the Waystar ID
    1. From the `Payer Description` get the text inside the last parenthesis
        
        ![Untitled](AMH2%20-%20EFT%20Insurance%20Payment%20Posting%20ddea2c7b736248b999644d69a526426f/Untitled%2010.png)
        
    2. Update `ws_id` = text
    3. Is `ws_id` in the `{run.ws_payors}` list?
        1. Y - Go to the next step
        2. N - Skip to the [Check for Last Records](https://www.notion.so/Check-for-Last-Records-57454aaf282f4987be00aed9ce063207?pvs=21) step
3. Create an EFT Object
    1. `{eft.check_num}` = ”ZERO PAY”
    2. `{eft.txn_date}` = `{run.date}` - 1
    3. `{eft.amount}` = “0.00”
    4. `{eft.desc1}` = `row.Payment Number`
    5. `{eft.desc2}` = “N/A”
    6. `{eft.desc3}` = “N/A”
    7. `{eft.835}`  = “Y”
    8. `{eft.waystar_id}`  = `ws_id`
    9. `{eft.matched}` = “”
    10. `{eft.835_file}` = “”
    11. `{eft.posted}` = “”
4. Append `{eft}` to `{run.efts}`
5. Run the [Download EOBs and 835s function](https://www.notion.so/Download-EOBs-and-835s-function-0043d2c6cf8f44ee9ab296947c3c2285?pvs=21) step
    1. `fn_eft` = `{eft}`
6. Check for Last Records
    1. Is this the last `row` in `rows`?
        1. Y - Go to the next step
        2. N - Go back to the [Loop through each row](https://www.notion.so/Loop-through-each-row-0bfb05b0eba442bd984cca93f387dd16?pvs=21) step and the next `row` in the loop
    2. Is this the last `page` in `pages`?
        1. Y - Skip to the [Macro 5.0 - Begin the Posting Process](https://www.notion.so/Macro-5-0-Begin-the-Posting-Process-263dbe3ac28f4ae3a8b8bb7955f49cb9?pvs=21) step
        2. N - Go back to the [Loop through each page](https://www.notion.so/Loop-through-each-page-c3b2db9684274088804597e638f268d1?pvs=21) step and the next `page` in the loop

## Macro 5.0 - Begin the Posting Process

<aside>
💡 **Loading Please Be Patient**

*Often, when developing in Traumasoft, you will need to wait for the window, page, process to complete.
You may or may not see a message.  Here is the html id = “app_mainView_loadingManager_loadingControl_popup”*

![Untitled](AMH2%20-%20EFT%20Insurance%20Payment%20Posting%20ddea2c7b736248b999644d69a526426f/Untitled%2011.png)

</aside>

### Macro 5.1 - Loop Through EFTs

1. Login to Traumasoft
    1. Navigate to Traumasoft Portal ⇒ [https://allmh.traumasoft.com/](https://allmh.traumasoft.com/)
    2. Log in using credentials from the [Get Bitwarden Credentials](https://www.notion.so/Get-Bitwarden-Credentials-3b1d40b95e55419388f6a18b9660560e?pvs=21) step
        1. Click on the `Login` button
2. Filter `{run.efts}` **WHERE** `{eft.posted}` ≠ “N”
3. For each `{eft}` in filtered `{run.efts}`:
    1. Go to the [Macro 5.2 - Fix Transaction IDs in 835 file](https://www.notion.so/Macro-5-2-Fix-Transaction-IDs-in-835-file-dbf16f03f92f485088afd781b4c5d429?pvs=21) step

### Macro 5.2 - Fix Transaction IDs in 835 file

1. Open the `{eft.835_file}` 
2. Search for any text that begins with `CLP|T5R` **OR** `CLP|1SR` **OR** `CLP|15R` **AND** replace with `CLP|TSR`
3. Save the altered file
4. Go to the [Macro 5.3 - Import the Batch](https://www.notion.so/Macro-5-3-Import-the-Batch-9ef4d101ab434fc58366099a35b4941f?pvs=21) step

### Macro 5.3 - Import the Batch

1. Navigate to the Batch Credits tab
    1. Click on the `V` (down arrow) in the `Billing` menu
    2. Select `Batch Credits` from the options
2. Import a new Batch
    1. Click on the `Open` button in the `New Batch` section
    2.  Select `From remittance file` radio button in the `Open Batch` window
    3. Click on the `Open` button
3. Complete Electronic Remittance Processing Form
    1. Select `ANSI 835` from the `Type of Remittance` options
    2. Select the 835
        1. Select `Local` radio button from `File location` options
        2. Click the `Choose File` button 
        3. Select `{eft.835_file}` 
        4. Click the `Open` button
    3. Select `Payment - Check` from the `Use this credit for payment` options
    4. Check the `Import allowances` checkbox 
    5. Select `Manual Contractual Allowance` from the `Import allowances` options
    6. Check the `Mandated Contractual Credit` checkbox
    7. Select `Manual Contractual Allowance` from the `Mandated Contractual Credit` options
    8. Check the `Replace automatic contractual...` checkbox
    9. Check the `Advance to next payor if non-zero balance` checkbox
    10. Select the `Do not advance on zero dollar payments` radio button
    11. Click the `OK` button
4. Check for Already Processed
    1. Does the `Traumasoft System Message` window contain “has already been processed”?
        1. Y - Go to the next step
        2. N - Skip to the [Check for Import Success](https://www.notion.so/Check-for-Import-Success-cb5ad341a07a479a86a1fa2d45b7932d?pvs=21) step
    2. Click on the `No` button
    3. Update EFT Record
        1. `{eft.posted}` = “E”
        2. `{eft.notes}` = “EFT has already been processed”
    4. Skip to the [Macro 10.0 - Complete the Run](https://www.notion.so/Macro-10-0-Complete-the-Run-561e1bda41944bb4998dde53d572005e?pvs=21) step
5. Check for Import Success
    1. Click on the `OK` button in the `Traumasoft System Message` window
    2. Wait for the file processing to finish
        
        ![Untitled](AMH2%20-%20EFT%20Insurance%20Payment%20Posting%20ddea2c7b736248b999644d69a526426f/Untitled%2012.png)
        
    3. Did the file import successfully?
        1. Y - Skip to the [Macro 5.4 - Open the Batch](https://www.notion.so/Macro-5-4-Open-the-Batch-78e53723dd9646a3866195dac8db9c3f?pvs=21) step
        2. N - Go to the next step
    4. Update EFT Record
        1. `{eft.posted}` = “N”
        2. `{eft.notes}` = “Error Importing EFT File”
    5. Skip to the [Macro 10.0 - Complete the Run](https://www.notion.so/Macro-10-0-Complete-the-Run-561e1bda41944bb4998dde53d572005e?pvs=21)  step

### Macro 5.4 - Open the Batch

1. Open the new Batch
    1. Click the `Open` button in the `New Batch` section
    2. Select `Existing Batch` from the `Open Batch` window
    3. Click the `Open` button
    4. From the `Batches` window, select the row where `Description` **CONTAINS** `{eft.835_file}`
    5. Click on the `OK` button
2. Update the Batch Information
    1. Click on the `Batch Info` button
    2. Type `{eft.835_file}` in the `Description` input
    3. Type `{eft.txn_date}` in the `Deposit Date` input
    4. Click on the `OK` button to close
3. Enter Target Totals
    1. Scroll down to the `Payment credits for all current batches` section *(very bottom of the page)*
    2. Check the `Target total:` checkbox
    3. Type `{eft.amount}` in the `Target total:` input
4. Check for Multiple Provider Payments
    1. For each `row` in `Batch Entries` table:
        1. Go to the next step
    2. Is the `Item` **IN** [“DENIALS/ADJ REASONS”, “Pe”,  “EXCEPTIONS”]?
        1.  Y - Continue to the next `row` in `Batch Entries` table
        2. N - Append to the `{eft.providers}` list 
        
        <aside>
        💡 **Batch Entry Table**
        
        ![Untitled](AMH2%20-%20EFT%20Insurance%20Payment%20Posting%20ddea2c7b736248b999644d69a526426f/Untitled%202.png)
        
        </aside>
        
5. Go to the [Macro 6.0 - Process Denials/Adjustments Reasons](https://www.notion.so/Macro-6-0-Process-Denials-Adjustments-Reasons-796cfc99f9d440bda25956d001704002?pvs=21) step

## Macro 6.0 - Process Denials/Adjustments Reasons

### Macro 6.1 - Check Denials/Adjustments Reasons

1. Is there `DENIALS/ADJ REASONS` item in the `Batch Entries` window?  *(See A in Image)*
    1. Y - Go to the next step
    2. N - Skip to the [Macro 7.0 - Process Provider Adjustments](https://www.notion.so/Macro-7-0-Process-Provider-Adjustments-cffaee9757264a6e813e055a128ef9f8?pvs=21) step
2. Click on `DENIALS/ADJ REASONS` item in the `Batch Entries` window
3. Loop through Denials and Adjustments
    1. For each `row` in `Denials/Adjustment Reasons` table:
        1. Go to the next step
4. Run the [Add Transaction Function](https://www.notion.so/Add-Transaction-Function-0110847b60c843e2846f629489ffc8cf?pvs=21) step
    1. `fn_eft` = `{eft}`
    2. `fn_txn_id` = `row.Transaction Id`
5. `current_txn` = `{txn}` from `{eft.txns}` **WHERE** `{txn.id}` = `row.Transaction Id`

### Macro 6.2 - Check Medicare Specific Rules

*Ask Marissa to check is it also VA?*

1. Is `{eft.payor}` = “Medicare Part B”?
    1. Y - Go to the next step
    2. N - Skip to the [Macro 6.3 - Check Next Payor and Current Payor](https://www.notion.so/Macro-6-3-Check-Next-Payor-and-Current-Payor-dda46efbdb7c4fb095a863c29828f293?pvs=21) step
2. Check Erroneous Codes
    1. Is `current_txn["status"]` = “Primary” **AND** (`Code` ≠ “CO45” **AND** `Code` ≠ “CO253”) **AND** `row.Current Payor` = `{eft.payor}`?
        1. Y - Take the following actions:
            - Get the `code` = `row.Code`
            - Get the `amt` = `row.Amount`
            - Go to the next step
        2. N - Skip to the [Macro 6.3 - Check Next Payor and Current Payor](https://www.notion.so/Macro-6-3-Check-Next-Payor-and-Current-Payor-dda46efbdb7c4fb095a863c29828f293?pvs=21) step
    2. Find the Matching Payment Row
        1. Click on the `Medicare Part B` item in the `Batch Entries` table  
        2. Is there a `row` in the `Credits` table **WHERE** `row.Transaction Id` = `current_txn["id"]` **AND** `row.Credit` = “Payment - Check” **AND** (`row.Specific Charge` = “BLS Disposables” **OR** `row.Specific Charge` = “Oxygen”)?
            - Y - Take the following actions:
                - Update `specific_charge` = `row.Specific Charge`  *(Note: Temporary Variable)*
                - Skip to the [Write Off the Charge](https://www.notion.so/Write-Off-the-Charge-d88b2b23889d40e5bfc6644f67266d0b?pvs=21) step
            - N - Go to the next step
        3. Take the following actions:
            - Run the [Change Payment to Denial Function](https://www.notion.so/Change-Payment-to-Denial-Function-c64445edccd846f8890026d9a65be982?pvs=21)
                - `fn_eft` = `{eft}`
                - `fn_txn_id` = `current_txn["id"]`
                - `fn_current_payor` = `current_txn["payor"]`
                - `fn_comments` =  “Set Payment on Denied Claim Schedule”
            - Run the [Add Transaction Change Function](https://www.notion.so/Add-Transaction-Change-Function-591970dcba214ec6a3f5eaa00f70326e?pvs=21)
                1. `fn_txn_id` = `current_txn["id"]`
                2. `fn_type` = “Adjustment/Denial”
                3. `fn_amt` = `row.Amount`
                4. `fn_note` = “Wrote Off `row.Code` for `specific_charge`.”
            - Skip to the [Macro 6.6 - Check for Last Denial and Adjustment](https://www.notion.so/Macro-6-6-Check-for-Last-Denial-and-Adjustment-f0339c6a78db47ddab83195ed65ca869?pvs=21) step
    3. Write Off the Charge
        1. Go back to the same `row` in `Denials/Adjustment Reasons` table
        2. Run the [Add Write Off Function](https://www.notion.so/Add-Write-Off-Function-400cf210bd2249b985967458e36373e2?pvs=21) 
            - `fn_eft` = `{eft}`
            - `fn_txn_id` = `current_txn["id"]`
            - `fn_amt` = `row.Amount`
            - `fn_applies_to` = `specific_charge`
            - `fn_comment` = “Wrote Off `row.Code` for `specific_charge`.”
        3. Run the [Add Transaction Change Function](https://www.notion.so/Add-Transaction-Change-Function-591970dcba214ec6a3f5eaa00f70326e?pvs=21) 
            1. `fn_txn_id` = `current_txn["id"]`
                1. `fn_type` = “Write Off”
            2. `fn_amt` = `row.Amount`
            3. `fn_note` = “Wrote Off `row.Code` for `specific_charge`.”

### Macro 6.3 - Check Next Payor and Current Payor

1. Is `row.Next Payor` = “” **AND** `row.Current Payor` ≠ `current_txn[”payor”]` **AND** `row.Type` = “Adj” **AND** (`row.Code` = “CO96” **OR** `row.Code` = “CO97”)? 
    1. Y - Go to the next step
    2. N - Skip to the [Macro 6.4 - Check Erroneous CO45 Adjustments](https://www.notion.so/Macro-6-4-Check-Erroneous-CO45-Adjustments-12e778e02b324b9c9607f6aefaac2352?pvs=21) step
2. Run the [Change Payment to Denial Function](https://www.notion.so/Change-Payment-to-Denial-Function-c64445edccd846f8890026d9a65be982?pvs=21) 
    1. `fn_eft` = `{eft}`
    2. `fn_txn_id` = `current_txn["id"]`
    3. `fn_current_payor` = `row.Current Payor`
    4. `fn_comments` =  “Set Payment on Denied Claim Schedule”
3. Run the [Add Transaction Change Function](https://www.notion.so/Add-Transaction-Change-Function-591970dcba214ec6a3f5eaa00f70326e?pvs=21) 
    1. `fn_txn_id` = `current_txn["id"]`
    2. `fn_type` = “Adjustment/Denial”
    3. `fn_amt` = `row.Amount`
    4. `fn_note` = “Set Payment on Denied Claim Schedule for Adjustment  = `row.Code`, Next Payor = “”, Current Payor = `row.Current Payor` and Payment Payor = `current_txn[”payor”]`.”

### Macro 6.4 - Check Erroneous CO45 Adjustments

1. Is `row.Type` = “Adj” **AND** `row.Code` = “CO45”?
    1. Y - Take the following actions:
        1. Get the `co45_amt` = `row.Amount`   *(Note: this is a temporary variable)*
        2. Go to the next step
    2. N - Skip to the [Macro 6.5 - Check for Insurance Paid to Patient](https://www.notion.so/Macro-6-5-Check-for-Insurance-Paid-to-Patient-c0b08d820bf14ea5a53f9527f555502d?pvs=21) step
2. Is there another `row` **WHERE** `Transaction Id` = `current_txn["id"]` 
**AND** `row.Type` = “Denial” **AND** (`row.Code` = “CO197” **OR** `row.Code` = “CO109”)  **
**OR** `row.Type` = “Adj” **AND** `row.Code` = “OA23”?
    1. Y - Take the following actions:
        1. Get the `erroneous_code` = `row.Code`   *(Note: this is a temporary variable)*
        2. Go to the next step
    2. N - Skip to the [Macro 6.5 - Check for Insurance Paid to Patient](https://www.notion.so/Macro-6-5-Check-for-Insurance-Paid-to-Patient-c0b08d820bf14ea5a53f9527f555502d?pvs=21) step
3. Loop through Providers
    1. For each `{provider}` in `{eft.providers}`:
        1. Click on the `{provider}` in the `Batch Entries` table
        2. Go to the next step
4. Loop through Credits
    1. For each `row` in the `Credits` table:
        1. Is ****`row.Transaction Id` = `current_txn["id"]` **AND** `row.Adj` = “CO45”?
        2. Y - Go to the next step
        3. N - Skip to the [Check for Last Records](https://www.notion.so/Check-for-Last-Records-4656bc72124748dea417685221a40af9?pvs=21) step
5. Delete the CO45 Adjustment
    1. Click on the `row` in the `Credits` table
    2. Click on the `Delete` button in the `Credits` row of the `Credit Processing` section
    3. Select the `Yes` button in the `Traumasoft System Message` window
    4. Run the [Add Transaction Change Function](https://www.notion.so/Add-Transaction-Change-Function-591970dcba214ec6a3f5eaa00f70326e?pvs=21) 
        1. `fn_txn_id` = `current_txn["id"]`
        2. `fn_type` = “Adjustment/Denial”
        3. `fn_amt` = `co45_amt`
        4. `fn_note` = “Deleted CO45 Adjustment where another row had `erroneous_code`.”
    5. Go to the next step
6. Check for Last Records
    1. Is this the last `row` in the `Credits` table?
        1. Y - Go to the next step
        2. N - Go back to the [Loop through Credits](https://www.notion.so/Loop-through-Credits-054364e6d4744386a8fe42ad5153ac57?pvs=21) step and the next `row` in the loop
    2. Is this the last `{provider}` in `{eft.providers}`?
        1. Y - Skip to the [Macro 6.6 - Check for Last Denial and Adjustment](https://www.notion.so/Macro-6-6-Check-for-Last-Denial-and-Adjustment-f0339c6a78db47ddab83195ed65ca869?pvs=21) step
        2. N - Go back to the [Loop through Providers](https://www.notion.so/Loop-through-Providers-28aedcbbdadd43b48db75bd134988d46?pvs=21) step and the next `{provider}` in the list

### Macro 6.5 - Check for Insurance Paid to Patient

1. Is there any row where `row.Code` = “OA100”?
    1. Y - Go to the next step
    2. N - Skip to the [Macro 6.6 - Check for Last Denial and Adjustment](https://www.notion.so/Macro-6-6-Check-for-Last-Denial-and-Adjustment-f0339c6a78db47ddab83195ed65ca869?pvs=21) step
2. Run the [Add Take Back Function](https://www.notion.so/Add-Take-Back-Function-0a13ed1c514c4e628fa12cf90d8299bb?pvs=21) 
    1. `fn_eft` = `{eft}`
    2. `fn_txn_id` = `current_txn["id"]`
    3. `fn_amt` = -`row.Amount`   *(Note: the amount should be negative)*
    4. `fn_applies_to` = `current_txn["payor"]`
    5. `fn_comment` = “`row.Code` - `row.Reason`"
3. Run the [Change Payment to Denial Function](https://www.notion.so/Change-Payment-to-Denial-Function-c64445edccd846f8890026d9a65be982?pvs=21) 
    1. `fn_eft` = `{eft}`
    2. `fn_txn_id` = `current_txn["id"]`
    3. `fn_current_payor` = `current_txn["payor"]`
    4. `fn_comments` =  “Set Payment on Denied Claim Schedule”
4. Run the [Add Transaction Change Function](https://www.notion.so/Add-Transaction-Change-Function-591970dcba214ec6a3f5eaa00f70326e?pvs=21) 
    1. `fn_txn_id` = `current_txn["id"]`
    2. `fn_type` = “Adjustment/Denial”
    3. `fn_amt` = `row.Amount`
    4. `fn_note` = “Adjusted off `row.Code` and set on Denial Schedule.”
5. Skip to the [Macro 6.6 - Check for Last Denial and Adjustment](https://www.notion.so/Macro-6-6-Check-for-Last-Denial-and-Adjustment-f0339c6a78db47ddab83195ed65ca869?pvs=21) step

### Macro 6.6 - Check for Last Denial and Adjustment

1. Is this the last row in  `row` in in `Denials/Adjustment Reasons` table?
    1. Y - Go to the [Macro 7.0 - Process Provider Adjustments](https://www.notion.so/Macro-7-0-Process-Provider-Adjustments-cffaee9757264a6e813e055a128ef9f8?pvs=21) step
    2. N - Go back to the [Loop through Denials and Adjustments](https://www.notion.so/Loop-through-Denials-and-Adjustments-2b7ce4d512ba4f0abe1fa315ec56e5f3?pvs=21) step and the next `row` in the loop

## Macro 7.0 - Process Provider Adjustments

### Macro 7.1 - Check Provider Adjustments

1. Is there a `Provider Adjustments` item in the `Batch Entries` window?  *(See C in Image)*
    1. Y - Go to the next step
    2. N - Skip to the [Macro 8.0 - Process Exceptions](https://www.notion.so/Macro-8-0-Process-Exceptions-11c8cf40686b428c94f3871c84de18b1?pvs=21) step
2. Click on `Provider Adjustments` item in the `Batch Entries` window
3. Loop through Provider Adjustments
    1. For each `row` in `ProviderAdjustments` table:
        1. Get the `pa_reason` = `row.Reason Code`
        2. Get the `pa_amt` = `row.Amount`
        *Note: remove the dollar sign but keep leading zeros, + or - and save as string*
        3. Go to the next step
4. Is the `pa_reason` = “L6”?
    1. Y - Take the following actions:
        1. Update `pa_ref` = `row.Reference Identification`
        2. Skip to the [Macro 7.2 - Check Interest](https://www.notion.so/Macro-7-2-Check-Interest-ffd0b160f61e4418b73aac161a771587?pvs=21) step
    2. N - Go to the next step
5. Is there only one number in the `row.Reference Identification` text?
    1. Y - Take the following actions:
        1. Update `{eft.posted}` = “E”
        2. Update `{eft.notes}` = “Partially Posted.  Provider Adjustments with no Transaction Id or Reference Identification.”
        3. Skip to the [Macro 7.6 - Check for Last Provider Adjustment](https://www.notion.so/Macro-7-6-Check-for-Last-Provider-Adjustment-5918dbc05dcb41e7b345793acee850d5?pvs=21) step
    2. N - Go to the next step
6. Run the [Add Transaction Function](https://www.notion.so/Add-Transaction-Function-0110847b60c843e2846f629489ffc8cf?pvs=21) 
    1. `fn_eft` = `{eft}`
    2. `fn_txn_id` = second number in `row.Reference Identification`
7. `current_txn` = `{txn}` from `{eft.txns}` **WHERE** `{txn.id}` = `row.Transaction Id`
8. Skip to the [Macro 7.3 - Check Forward Balance](https://www.notion.so/Macro-7-3-Check-Forward-Balance-f5727515f60b42e8a38cf2968ccf92c9?pvs=21) step

### Macro 7.2 - Check Interest

1. Get Transaction Number
    1. Open the corresponding EOB “DEP `{eft.txn_date}`_`{eft.waystar_id}`_`{eft.amount}`.pdf”
    2. Search for “INTEREST -`pa_amt`"   *(Note should be positive number)*
    3. Get the corresponding `ACNT:`
        
        ![Untitled](AMH2%20-%20EFT%20Insurance%20Payment%20Posting%20ddea2c7b736248b999644d69a526426f/Untitled%2013.png)
        
    4. Run the [Add Transaction Function](https://www.notion.so/Add-Transaction-Function-0110847b60c843e2846f629489ffc8cf?pvs=21) 
        1. `fn_eft` = `{eft}`
        2. `fn_txn_id` = `ACNT:`
    5. `current_txn` = `{txn}` from `{eft.txns}` **WHERE** `{txn.id}` = `row.Transaction Id`
    6. Run the [Add Interest Function](https://www.notion.so/Add-Interest-Function-be60a1a333d84b409016df9cb9f1b3c5?pvs=21) 
        1. `fn_eft` = `{eft}`
        2. `fn_txn_id` = `current_txn["id"]`
        3. `fn_interest` = -`pa_amt`     *(Note should be positive number)*
        4. `fn_comment` = “Added Provider Adjustment (L6) Interest”
    7. Run the [Add Transaction Change Function](https://www.notion.so/Add-Transaction-Change-Function-591970dcba214ec6a3f5eaa00f70326e?pvs=21) 
        1. `fn_txn_id` = `current_txn["id"]`
        2. `fn_type` = “Interest”
        3. `fn_amt` = `pa_amt`     *(Note should be negative number)*
        4. `fn_note` = “Added Provider Adjustment (L6) Interest”
    8. Run the [Add EOB to Patient Account Function](https://www.notion.so/Add-EOB-to-Patient-Account-Function-c8462065b37f4a15bf7bdaedc71a3f8d?pvs=21)
        1. `fn_eft` = `{eft}`
        2. `fn_leg_id` = `current_txn["id"]`

### Macro 7.3 - Check Forward Balance

1. Is the `pa_reason` = “FB”?
    1. Y - Go to the next step
    2. N - Skip to the [Macro 7.4 - Check Medicare Take Backs](https://www.notion.so/Macro-7-4-Check-Medicare-Take-Backs-0c87d08ebb154eda8d91e398984db717?pvs=21) step
2. Create temporary variable `sum` = 0
3. Loop through Providers
    1. For each `{provider}` in `{eft.providers}`:
        1. Click on the `{provider}` in the `Batch Entries` table
        2. Go to the next step
4. Loop through Credits
    1. For each `row` in the `Credits` table:
        1. Go to the next step
    2. Is `current_txn["id"]` = `Transaction Id` **AND** `Credit` = “Payment - Check”?
        1. Y - Subtract `Amount` from `sum`
        2. N - Go back to the next `row` in the loop
5. Check for Last Records
    1. Is this the last `row` in the `Credits` table?
        1. Y - Go to the next step
        2. N - Go back to the [Loop through Credits](https://www.notion.so/Loop-through-Credits-410fa1ab86094590b425d4fd88ad75fa?pvs=21) and the next `row` in the loop
    2. Is this the last `{provider}` in `{eft.providers}`?
        1. Y - Go to the next step
        2. N - Go back to the [Loop through Providers](https://www.notion.so/Loop-through-Providers-096426f5b3514843943122796099eb0c?pvs=21) step and the next `{provider}` in the list
6. Is `sum` = 0 **OR** is `sum` ≠ `pa_amt`?
    1. Y - Take the following actions:
        1. Update EFT Record
            - `{eft.posted}` = “E”
            - Append `{eft.notes}` = “Partially Posted.  Could not match total of payments to Forward Balance.”
        2. Skip to the [Macro 7.6 - Check for Last Provider Adjustment](https://www.notion.so/Macro-7-6-Check-for-Last-Provider-Adjustment-5918dbc05dcb41e7b345793acee850d5?pvs=21) step
    2. N - Go to the next step
7. Loop through Providers and Credits again
    1. For each `{provider}` in `{eft.providers}`:
        1. Click on the `{provider}` in the `Batch Entries` table
    2. For each `row` in the `Credits` table **WHERE** `row.Transaction Id` = `current_txn["id"]` **AND** `Credit` = “Payment - Check”:
        1. Click on the `row` to highlight
        2. Run the [Zero Out Forward Balance Function](https://www.notion.so/Zero-Out-Forward-Balance-Function-6a3b759928234c2982773122a8917674?pvs=21) 
            1. `{fn_eft}` = `{eft}`
8. Run the [Add Transaction Change Function](https://www.notion.so/Add-Transaction-Change-Function-591970dcba214ec6a3f5eaa00f70326e?pvs=21) 
    1. `fn_txn_id` = `current_txn["id"]`
    2. `fn_type` = “Provider Adjustment”
    3. `fn_amt` = `pa_amt`
    4. `fn_note` = “Zeroed Out Forwarding Balance”
9. Run the [Add EOB to Patient Account Function](https://www.notion.so/Add-EOB-to-Patient-Account-Function-c8462065b37f4a15bf7bdaedc71a3f8d?pvs=21)
    1. `fn_eft` = `{eft}`
    2. `fn_leg_id` = `current_txn["id"]`
10. Go to the [Macro 7.6 - Check for Last Provider Adjustment](https://www.notion.so/Macro-7-6-Check-for-Last-Provider-Adjustment-5918dbc05dcb41e7b345793acee850d5?pvs=21) step

### Macro 7.4 - Check Medicare Take Backs

1. Is the `pa_reason` ≠ “FB”?
    1. Y - Go to the next step
    2. N - Skip to the [Macro 7.6 - Check for Last Provider Adjustment](https://www.notion.so/Macro-7-6-Check-for-Last-Provider-Adjustment-5918dbc05dcb41e7b345793acee850d5?pvs=21) step
2. Is any `{provider}` in `{eft.providers}` = “Medicare Part B”?
    1. Y - Go to the next step
    2. N - Skip to the [Macro 7.5 - Check Other Take Backs](https://www.notion.so/Macro-7-5-Check-Other-Take-Backs-653104cf58d545678e87d5354a14e267?pvs=21) step
3. Navigate to the Call Properties in a new Browser Window
    1. `url` = “[https://allmh.traumasoft.com/main.php?a=billing:validation/main&popup=1#id:](https://allmh.traumasoft.com/main.php?a=billing:validation/main&popup=1#id:)`{current_txn["leg_id"]}`”
    2. Click on the `Billing` tab
    3. Click on the `Enable Editing` button
        
        ![Untitled](AMH2%20-%20EFT%20Insurance%20Payment%20Posting%20ddea2c7b736248b999644d69a526426f/Untitled%2014.png)
        
4. Loop Through Credits
    1. For each `credit` in `Credits` table:
        1. Go to the next step
    2. Is `Credit` column = “Payment - Check”?
        1. Y - Go to the next step
        2. N - Continue to next `credit` in `Credits` table
    3. Click on the `credit` row
    4. Click on `Edit` button
    5. Does `Credit applies to ...` option selected contain “Mileage”?
        1. Y - Update `mileage_amt` = `Amount` in the `Edit Credit` window
        2. N - Update `base_amt` = `Amount` in the `Edit Credit` window
    6. Click on the `Cancel` button
        
        ![Untitled](AMH2%20-%20EFT%20Insurance%20Payment%20Posting%20ddea2c7b736248b999644d69a526426f/Untitled%2015.png)
        
5. Update Interest
    1. `interest_amt` = `pa_amt` - `mileage_amt` - `base_amt`
6. Run the [Add Take Back Function](https://www.notion.so/Add-Take-Back-Function-0a13ed1c514c4e628fa12cf90d8299bb?pvs=21) 
    1. `fn_eft` = `{eft}`
    2. `fn_txn_id` = `current_txn["id"]`
    3. `fn_amt` = -`mileage_amt`   *(Note: Should be negative)*
    4. `fn_applies_to` = “mileage”
    5. `fn_comment` = “Added Provider Adjustment (`pa_reason`) Take Back.”
7. Run the [Add Take Back Function](https://www.notion.so/Add-Take-Back-Function-0a13ed1c514c4e628fa12cf90d8299bb?pvs=21) 
    1. `fn_eft` = `{eft}`
    2. `fn_txn_id` = `current_txn["id"]`
    3. `fn_amt` = -`base_amt`   *(Note: Should be negative)*
    4. `fn_applies_to` = “base rate”
    5. `fn_comment` = “Added Provider Adjustment (`pa_reason`) Take Back.”
8. Is `interest_amt` = 0?
    1. Y - Skip to the [Complete Take Back Process](https://www.notion.so/Complete-Take-Back-Process-2ac8d72f3c104a1e8d59c075e171e70e?pvs=21) step
    2. N - Go to the next step
9. Run the [Add Interest Function](https://www.notion.so/Add-Interest-Function-be60a1a333d84b409016df9cb9f1b3c5?pvs=21) 
    1. `fn_eft` = `{eft}`
    2. `fn_txn_id` = `current_txn["id"]`
    3. `fn_interest` = -`interest_amt`   *(Note: Should be negative amount)*
    4. `fn_comment` = “Added Provider Adjustment (`pa_reason`) Take Back.”
10. Complete Take Back Process
    1. Run the [Add Transaction Change Function](https://www.notion.so/Add-Transaction-Change-Function-591970dcba214ec6a3f5eaa00f70326e?pvs=21) 
        1. `fn_txn_id` = `current_txn["id"]`
        2. `fn_type` = “Provider Adjustment”
        3. `fn_amt` = `pa_amt`
        4. `fn_note` = “Added Provider Adjustment (`pa_reason`) Take Back to Mileage, Base Rate and Interest.”
    2. Run the [Add EOB to Patient Account Function](https://www.notion.so/Add-EOB-to-Patient-Account-Function-c8462065b37f4a15bf7bdaedc71a3f8d?pvs=21)
        1. `fn_eft` = `{eft}`
        2. `fn_leg_id` = `current_txn["id"]`
11. Skip to the [Macro 7.6 - Check for Last Provider Adjustment](https://www.notion.so/Macro-7-6-Check-for-Last-Provider-Adjustment-5918dbc05dcb41e7b345793acee850d5?pvs=21) step

### Macro 7.5 - Check Other Take Backs

1. Run the [Add Take Back Function](https://www.notion.so/Add-Take-Back-Function-0a13ed1c514c4e628fa12cf90d8299bb?pvs=21) 
    1. `fn_eft` = `{eft}`
    2. `fn_txn_id` = `current_txn["id"]`
    3. `fn_amt` = -`pa_amt`
    4. `fn_applies_to` = “”
    5. `fn_comment` = “Added Provider Adjustment (`pa_reason`) Take Back.”
2. Run the [Add Transaction Change Function](https://www.notion.so/Add-Transaction-Change-Function-591970dcba214ec6a3f5eaa00f70326e?pvs=21) 
    1. `fn_txn_id` = `current_txn["id"]`
    2. `fn_type` = “Provider Adjustment”
    3. `fn_amt` = `pa_amt`
    4. `fn_note` = “Added Provider Adjustment (`pa_reason`) Take Back.”
3. Run the [Add EOB to Patient Account Function](https://www.notion.so/Add-EOB-to-Patient-Account-Function-c8462065b37f4a15bf7bdaedc71a3f8d?pvs=21)
    1. `fn_eft` = `{eft}`
    2. `fn_leg_id` = `current_txn["id"]`
4. Go to the [Macro 7.6 - Check for Last Provider Adjustment](https://www.notion.so/Macro-7-6-Check-for-Last-Provider-Adjustment-5918dbc05dcb41e7b345793acee850d5?pvs=21) step

### Macro 7.6 - Check for Last Provider Adjustment

1. Is this the last row in  `row` in `ProviderAdjustments` table?
    1. Y - Go to the [Macro 8.0 - Process Exceptions](https://www.notion.so/Macro-8-0-Process-Exceptions-11c8cf40686b428c94f3871c84de18b1?pvs=21) step
    2. N - Go back to the [Loop through Provider Adjustments](https://www.notion.so/Loop-through-Provider-Adjustments-3ae16ade98f54b0eb06667f0287ee969?pvs=21) step and the next `row` in the loop

## Macro 8.0 - Process Exceptions

### Macro 8.1 - Check Exceptions

1. Is there `EXCEPTIONS` item in the `Batch Entries` window?  *(See B in Image)*
    1. Y - Go to the next step
    2. N - Skip to the [Macro 9.0 - Process Payments](https://www.notion.so/Macro-9-0-Process-Payments-939bb520d353497ab6ce7f0a3eb1580c?pvs=21) step
2. Click on `EXCEPTIONS` item in the `Batch Entries` window
3. Loop through Exceptions
    1. For each `row` in `Exceptions` table:  
        1. Go to the next step
4. Does the `row.Transaction ID` begin with “TSR”?
    1. Y - Go to the next step
    2. N - Take the following actions:
        1. Update `txn_id` = `row.Transaction ID`
        2. Skip to the [Get Transaction Information](https://www.notion.so/Get-Transaction-Information-c533a551c5b8460e8e3654f2b850ebdb?pvs=21) step
5. Get the Run ID
    1. Get `run_id` = `row.Transaction ID` without “TSR” in beginning and add - before last 2 digits
    ex:	TSR16142123 ⇒ 161421-23
    2. In a new browser tab navigate to “https://allmh.traumasoft.com/main.php?a=billing:workflow/main”
    3. Select `Run#` from the `Select One` options
    4. Type `run_id` into the `Transaction ID` input
    5. Click on the `Search` button
6. Is there more than one row?
    1. Y - Take the following Actions:
        1. Update EFT Record
            - `fn_eft["posted"]` = “P”
            - `fn_eft["notes"]` = “Partially Posted. Multiple rows Run Number search. ID: `row.Transaction ID`”
        2. End Function
    2. N - Go to the next step
7. Get the Transaction ID
    1. Double-Click on the row
    2. Click on the `Billing Actions` button in the `Call Properties: Allegiance` window
    3. Select the `Transaction Ids` option
    4. Update `txn_id` = the last Transaction ID in the list (lowest one in the list)
    5. Close the `Call Properties: Allegiance` window
    6. Close the Browser Tab
8. Get Transaction Information
    1. Run the [Add Transaction Function](https://www.notion.so/Add-Transaction-Function-0110847b60c843e2846f629489ffc8cf?pvs=21) 
        1. `fn_eft` = `{eft}`
        2. `fn_txn_id` = `txn_id`
9. `current_txn` = `{txn}` from `{eft.txns}` **WHERE** `{txn.id}` = `row.Transaction Id`

### Macro 8.2 - Check for Deleted Charges

1. Does `row.Exceptions` contain “Can't find matching charge on trip for this credit”?
    1. Y - Go to the next step
    2. N - Go to the [Macro 8.3 - Check for Missing Payor](https://www.notion.so/Macro-8-3-Check-for-Missing-Payor-7ae01f46eda54a84b225547c04b77dc2?pvs=21) step
2. Run [Change Payment to Denial Function](https://www.notion.so/Change-Payment-to-Denial-Function-c64445edccd846f8890026d9a65be982?pvs=21) 
    1. `fn_eft` = `{eft}`
    2. `fn_txn_id` = `current_txn["id"]`
    3. `fn_current_payor` = `current_txn["payor"]`
    4. `fn_comments` = “Set Payment on Denied Claim Schedule for Missing Charge on Trip.”
        
        ![Untitled](AMH2%20-%20EFT%20Insurance%20Payment%20Posting%20ddea2c7b736248b999644d69a526426f/Untitled%2016.png)
        
3. Run the [Add Transaction Change Function](https://www.notion.so/Add-Transaction-Change-Function-591970dcba214ec6a3f5eaa00f70326e?pvs=21) 
    1. `fn_txn_id` = `current_txn["id"]`
    2. `fn_type` = “Exception”
    3. `fn_amt` = `row.Amount`
    4. `fn_note` = “Set Payment on Denied Claim Schedule for Missing Charge on Trip.”

### Macro 8.3 - Check for Missing Payor

1. Does `row.Exceptions` contain “Patient does not have a payor with remit code?
    1. Y - Go to the next step
    2. N - Go to the [Macro 8.4 - Check for Last Exception](https://www.notion.so/Macro-8-4-Check-for-Last-Exception-fd24386a410d47e98eb9525b3f561b36?pvs=21) step
2. Run the [Add New Payor to Patient Record Function](https://www.notion.so/Add-New-Payor-to-Patient-Record-Function-15821902e0aa4a3480699d962927b6e7?pvs=21) 
    1. `fn_txn` = `current_txn`
    2. `fn_eft` = `{eft}`
    3. `fn_amt` = `row.Amount`
3. Run the [Add Transaction Change Function](https://www.notion.so/Add-Transaction-Change-Function-591970dcba214ec6a3f5eaa00f70326e?pvs=21) 
    1. `fn_txn_id` = `current_txn["id"]`
    2. `fn_type` = “Exception”
    3. `fn_amt` = `row.Amount`
    4. `fn_note` = “Added Missing Payor to Patient Record”
        
        ![Untitled](AMH2%20-%20EFT%20Insurance%20Payment%20Posting%20ddea2c7b736248b999644d69a526426f/Untitled%2017.png)
        

### Macro 8.4 - Check for Last Exception

1. Is this the last row in  `row` in `Exceptions` table?
    1. Y - Go to the [Macro 9.0 - Process Payments](https://www.notion.so/Macro-9-0-Process-Payments-939bb520d353497ab6ce7f0a3eb1580c?pvs=21) step
    2. N - Go back to the [Loop through Exceptions](https://www.notion.so/Loop-through-Exceptions-188fb2429c35439ba4ab49269bbc7d85?pvs=21) step and the next `row` in the loop

## Macro 9.0 - Process Payments

### Macro 9.1 - Loop Through Payments

1. Loop through Providers
    1. For each `{provider}` in `{eft.providers}`:
        1. Click on the `{provider}` in the `Batch Entries` table
        2. Go to the next step
2. Loop through Credits
    1. For each `row` in the `Credits` table:
        1. Go to the next step
3. Run the [Add Transaction Function](https://www.notion.so/Add-Transaction-Function-0110847b60c843e2846f629489ffc8cf?pvs=21) 
    1. `fn_eft` = `{eft}`
    2. `fn_txn_id` = `row.Transaction Id`
4. `current_txn` = `{txn}` from `{eft.txns}` **WHERE** `{txn.id}` = `row.Transaction Id`

### Macro 9.2 - Check Negative Adjustments

1. Is `row.Amount` < 0 **AND** `row.Amount` + `row.New Balance` = 0 **AND** `row.Code` = “CO45”?
    1. Y - Go to the next step
    2. N - Skip to the [Macro 9.3 - Check Advanced to Next Payor](https://www.notion.so/Macro-9-3-Check-Advanced-to-Next-Payor-f5e13b1fbf2f43b2b47435ac19e0c3d9?pvs=21) step
2. Delete the CO45 Adjustment
    1. Click on the `row` in the `Credits` table
    2. Click on the `Delete` button in the `Credits` row of the `Credit Processing` section
    3. Select the `Yes` button in the `Traumasoft System Message` window
    4. Run the [Add Transaction Change Function](https://www.notion.so/Add-Transaction-Change-Function-591970dcba214ec6a3f5eaa00f70326e?pvs=21) 
        1. `fn_txn_id` = `current_txn["id"]`
        2. `fn_type` = “Adjustment/Denial”
        3. `fn_amt` = `row.Amount`
        4. `fn_note` = “Deleted CO45 Adjustment on Negative Adjustment Row.”
    5. Skip to the [Macro 9.4 - Check for Last Providers](https://www.notion.so/Macro-9-4-Check-for-Last-Providers-2fa9b82d792a45bc8943c816d27c9585?pvs=21) step

### Macro 9.3 - Check Advanced to Next Payor

1. Is `row.Credit` = “Payment - Check”?
    1. Y - Go to the next step
    2. N - Skip to the [Macro 9.4 - Check for Last Providers](https://www.notion.so/Macro-9-4-Check-for-Last-Providers-2fa9b82d792a45bc8943c816d27c9585?pvs=21) step
2. Is `current_txn["crossover"]` = “”?
    1. Y - Skip to the [Macro 9.4 - Check for Last Providers](https://www.notion.so/Macro-9-4-Check-for-Last-Providers-2fa9b82d792a45bc8943c816d27c9585?pvs=21) step
    2. N - Go to the next step
3. Click on the `row` in the `Credits` table
4. Click on the `Edit` button in the `Credit` row of the `Credit Processing` form   *(See Credit Processing Form)*
5. Navigate to the `Trip` section of the `Edit Credit` window
6. Get Traumasoft Crossover Payor
    1. Navigate to the `Crossover Payors` tab of the `AMH2 Mapping.xlsx` [HERE](https://allegianceambulance-my.sharepoint.com/:x:/r/personal/thoughtful_allmh_com/_layouts/15/Doc.aspx?sourcedoc=%7BCBCE1B8B-6912-4141-A453-437B3E6737F9%7D&file=AMH2%20Mapping.xlsx&action=default&mobileredirect=true)
    2. Filter `CROSSOVER` = `current_txn["crossover"]`
    3. Is there a row **AND** `TRAUMASOFT PAYOR` ≠ “”?
        1. Y - Take the following actions:
            - Update `current_txn["crossover"]` = corresponding `TRAUMASOFT PAYOR`
            - Skip to the [Check Existing Payors](https://www.notion.so/Check-Existing-Payors-2b05d5fc313f4d22b6bc4adf36808d3f?pvs=21) step
        2. N - Go to the next step
    4. Add new row to the `Crossover Payors` tab
        1. `CROSSOVER` = `current_txn["crossover"]`
        2. `TRAUMASOFT PAYOR` = “”
    5. Run the [Add Transaction Change Function](https://www.notion.so/Add-Transaction-Change-Function-591970dcba214ec6a3f5eaa00f70326e?pvs=21) 
        1. `fn_txn_id` = `current_txn["id"]`
        2. `fn_type` = “Exception”
        3. `fn_amt` = `row.Amount`
        4. `fn_note` = “Crossover Payor Not Found in the AMH2 Mapping File”
    6. Skip to the [Macro 9.4 - Check for Last Providers](https://www.notion.so/Macro-9-4-Check-for-Last-Providers-2fa9b82d792a45bc8943c816d27c9585?pvs=21) step
7. Check Existing Payors
    1. Is there any `payor` in the `Next Payor:` options = `current_txn["crossover"]`?
        1. Y - Skip to [Update Crossover Payor](https://www.notion.so/Update-Crossover-Payor-7a9ff142261a4334b700e9a6a5e205ac?pvs=21) step
        2. N - Go to the next step
8. Add Crossover Payor
    1. Click on the `row.Name` hyperlink
    2. Get the `dos` = `row.Date`
    3. Navigate to the `Payors` section in the new browser tab
    4. Click on the `[New Payor]` hyperlink
    5. Type `current_txn["crossover"]` in the `Payor` input of the `Add Payor` window
    6. Double-click the matching `Payor Name` in the `Select Payor` window
    7. Type “2” in the `Order` input **IF** `current_txn["status"]` = “Primary” **ELSE** Type “3”
    8. Click on the `OK` button
    9. Click on the `OK` button in the `Traumasoft System Message` window
9. Check Eligibility
    1. Click on the row in the `Payors` table **WHERE** `current_txn["crossover"]` = `Payor Name`
    2. Get Eligibility Information
        1. Click on the `Check Eligibility` hyperlink in the `actions` column
        2. Select `Manual` from the options in the `Select Eligibility Service` window
        3. Type `dos` in the `Date:` input
        4. Click on the `OK` button
        5. Click on the `Check Eligibility` hyperlink again
        6. Select `Payor Logic` from the options in the `Select Eligibility Service` window
        7. Click on the `OK` button
    3. Is `Status:` = “Active Coverage” in the `View Results` window?
        1. Y - Take the following actions:
            - Get the `subscriber_id` = `Subscriber ID:`
            - Click on the `Close` button
            - Skip to the [Add Subscriber ID](https://www.notion.so/Add-Subscriber-ID-97c0ebc5ee314c67b24d1560eebcfe05?pvs=21) step
        2. N - Go to the next step
    4. Run the [Add Transaction Change Function](https://www.notion.so/Add-Transaction-Change-Function-591970dcba214ec6a3f5eaa00f70326e?pvs=21) 
        1. `fn_txn_id` = `current_txn["id"]`
        2. `fn_type` = “Exception”
        3. `fn_amt` = `row.Amount`
        4. `fn_note` = “Crossover Payor is Not Showing Active in Payor Logic.”
    5. Skip to the [Macro 9.4 - Check for Last Providers](https://www.notion.so/Macro-9-4-Check-for-Last-Providers-2fa9b82d792a45bc8943c816d27c9585?pvs=21) step
10. Add Subscriber ID
    1. Click on the `Edit` hyperlink in the `actions` column
    2. Type `subscriber_id` in the `Identification:` input
    3. Click on the `OK` button
    4. Click on the `OK` button in the `Traumasoft System Message` window
11. Close the `Edit Patient` window
12. Run the [Add Transaction Change Function](https://www.notion.so/Add-Transaction-Change-Function-591970dcba214ec6a3f5eaa00f70326e?pvs=21) 
    1. `fn_txn_id` = `current_txn["id"]`
    2. `fn_type` = “Added Payor”
    3. `fn_amt` = `row.Amount`
    4. `fn_note` = “Added Crossover Payor to Patient Record.”
13. Go back to the original tab and the `row` in the `Credits` table
14. Run the [Update Next Payor Function](https://www.notion.so/Update-Next-Payor-Function-07bacce4f1c845bbae6dbfdb654f070d?pvs=21) 
    1. `fn_crossover` = `current_txn["crossover"]`
    2. `fn_comments` = “Updated Next Payor to Crossover Payor.”
15. Run the [Add Transaction Change Function](https://www.notion.so/Add-Transaction-Change-Function-591970dcba214ec6a3f5eaa00f70326e?pvs=21) 
    1. `fn_txn_id` = `current_txn["id"]`
    2. `fn_type` = “Updated Next Payor”
    3. `fn_amt` = `row.Amount`
    4. `fn_note` = “Updated Next Payor to Crossover Payor.”
16. Skip to the [Macro 9.4 - Check for Last Providers](https://www.notion.so/Macro-9-4-Check-for-Last-Providers-2fa9b82d792a45bc8943c816d27c9585?pvs=21) step

### Macro 9.4 - Check for Last Providers

1. Check for Posted No Changes
    1. Is `current_txn["changes"]` an empty list?
        1. Y - Go to the next step
        2. N - Skip to the [Check for Last Records](https://www.notion.so/Check-for-Last-Records-674335e8a50c4522b51fed75581a1250?pvs=21) step
    2. Run the [Add Transaction Change Function](https://www.notion.so/Add-Transaction-Change-Function-591970dcba214ec6a3f5eaa00f70326e?pvs=21) 
        1. `fn_txn_id` = `current_txn["id"]`
        2. `fn_type` = “Payment”
        3. `fn_amt` = `row.Amount`
        4. `fn_note` = “Posted Payment with No Changes.”
2. Check for Last Records
    1. Is this the last `row` in the `Credits` table?
        1. Y - Go to the next step
        2. N - Go back to the [Loop through Credits](https://www.notion.so/Loop-through-Credits-a3194de63087490c9c21adf6d48c63dc?pvs=21) step and the next `row` in the loop
    2. Is this the last `{provider}` in `{eft.providers}`?
        1. Y - Go to the next step
        2. N - Go back to the [Loop through Providers](https://www.notion.so/Loop-through-Providers-4d09ec312b734395b4e497b15c34761a?pvs=21) step and the next `{provider}` in the list

### Macro 9.5 - Post the Batch

1. Is the “Post Batches” option selected from the [Macro 1.1 - Get Selections from Empower](https://www.notion.so/Macro-1-1-Get-Selections-from-Empower-8d0f2f81cacb4fa68db8b5e8604739d8?pvs=21) step?
    1. Y - Go to the next step
    2. N - Skip to the [Macro 9.6 - Complete the Posting Process](https://www.notion.so/Macro-9-6-Complete-the-Posting-Process-cb07cd0c1a434887933b89979b9a0ede?pvs=21) step
2. Click on the `Post Batch` button
    
    ![Untitled](AMH2%20-%20EFT%20Insurance%20Payment%20Posting%20ddea2c7b736248b999644d69a526426f/Untitled%2018.png)
    
3. Is “Target total and Amount to Post do not match” in the `Traumasoft System Message` window?
    1. Y - Take the following actions:
        1. Click on the `No` button
        2. Update `{eft.posted}` = “N”
        3. Append to beginning of `{eft.notes}` ⇒ “Target Total and Amount to Post Do Not Match.”
            
            ![Untitled](AMH2%20-%20EFT%20Insurance%20Payment%20Posting%20ddea2c7b736248b999644d69a526426f/Untitled%2019.png)
            
    2. N - Go to the next step
4. Is there a `Duplicate Items Found` window?
    1. Y - Click on the `Post With Duplicates` button
    2. N - Go to the next step
        
        ![Untitled](AMH2%20-%20EFT%20Insurance%20Payment%20Posting%20ddea2c7b736248b999644d69a526426f/Untitled%2020.png)
        
5. Update Batch Descriptions and Dates
    1. Type `{eft.txn_date}` in `Deposit Date:` input
        
        ![Untitled](AMH2%20-%20EFT%20Insurance%20Payment%20Posting%20ddea2c7b736248b999644d69a526426f/Untitled%2021.png)
        
    2. Leave all other fields as default
    3. Click on the `OK` button
6. Click on the `YES` button in the `Traumasoft System Message` window
    
    ![Untitled](AMH2%20-%20EFT%20Insurance%20Payment%20Posting%20ddea2c7b736248b999644d69a526426f/Untitled%2022.png)
    
7. Go to the [Macro 9.6 - Complete the Posting Process](https://www.notion.so/Macro-9-6-Complete-the-Posting-Process-cb07cd0c1a434887933b89979b9a0ede?pvs=21) step

<aside>
💡 **IMPORTANT DEVELOPMENT NOTE**

You **CAN** go all the way to the end on a batch **YOU** imported for development purposes.  To **DEPOST** a batch follow these directions:

1. Click on `Open` button in the `Batches` menu
2. Select `Previously posted batch` radio button in the `Open Batch` window
    
    ![Untitled](AMH2%20-%20EFT%20Insurance%20Payment%20Posting%20ddea2c7b736248b999644d69a526426f/Untitled%2023.png)
    
3. Click on the `Open` button
4. Select Posted Batches Window Options
    1. Select `Batch Poster`  options
    2. Type `Thoughtful` in the input
    3. Click on the `Search` button
    4. Select the row of the batch you want to Depost
    5. Click on the `OK` button
        
        ![Untitled](AMH2%20-%20EFT%20Insurance%20Payment%20Posting%20ddea2c7b736248b999644d69a526426f/Untitled%2024.png)
        
5. When it finishes loading
6. Click on the `De-Post Batch` button
    
    ![Untitled](AMH2%20-%20EFT%20Insurance%20Payment%20Posting%20ddea2c7b736248b999644d69a526426f/Untitled%2025.png)
    
</aside>

### Macro 9.6 - Complete the Posting Process

1. Is this the last `{eft}` in `{run.efts}`:
    1. Y - Go to the [Macro 10.0 - Complete the Run](https://www.notion.so/Macro-10-0-Complete-the-Run-561e1bda41944bb4998dde53d572005e?pvs=21) step
    2. N - Go back to the [Macro 5.1 - Loop Through EFTs](https://www.notion.so/Macro-5-1-Loop-Through-EFTs-0754e7dc4e964fd9bb8a739baa7a9a18?pvs=21) and the next `{eft}` in the loop

## Macro 10.0 - Complete the Run

### Macro 10.1 - Update Bank Deposit Master

1. Loop Through All EFTs
    1. For each `{eft}` in `{run.efts}`:
        1. Go to the next step
2. Update Bank Deposit Master
    1. Navigate to the `Bank Deposit Master.xlsx` file in Sharepoint
    2. Is there a matching row where …
    `CHECK NUM` = `{eft.check_num}`, `TXN DATE` = `{eft.txn_date}`, `AMOUNT` = `{eft.amount}`, 
    `DESC1` = `{eft.desc1}`, `DESC2` = `{eft.desc2}`, `DESC3` = `{eft.desc3}`, `835` = `{eft.835}` ?
        1. Y - Go to the next step
        2. N - Skip to the [Add New Row](https://www.notion.so/Add-New-Row-c66700beb4a4472988fb9568f328d36c?pvs=21) step
    3. Update Existing Row
        1. `WAYSTAR ID` = `{eft.waystar_id}`
        2. `MATCHED` = `{eft.matched}`
        3. `835 FILE` = `{eft.835_file}`
        4. `POSTED` = `{eft.posted}`
        5. `NOTES` = `{eft.notes}`
    4. Add New Row
        1. `CHECK NUM` = `{eft.check_num}`
        2. `TXN DATE` = `{eft.txn_date}`
        3. `AMOUNT` = `{eft.amount}`
        4. `DESC1` = `{eft.desc1}`
        5. `DESC2` = `{eft.desc2}`
        6. `DESC3` = `{eft.desc3}`
        7. `835` = `{eft.835}`
        8. `WAYSTAR ID` = `{eft.waystar_id}`
        9. `MATCHED` = `{eft.matched}`
        10. `835 FILE` = `{eft.835_file}`
        11. `POSTED` = `{eft.posted}`
        12. `NOTES` = `{eft.notes}`
3. Check for Last Record
    1. Is this the last `{eft}` in `{run.efts}`:
        1. Y - Go to the [Macro 10.2 - Update Records for Data Capture](https://www.notion.so/Macro-10-2-Update-Records-for-Data-Capture-91c5dfcd68e14f21884ff029f123162b?pvs=21) step
        2. N - Go back to the [Loop Through All EFTs](https://www.notion.so/Loop-Through-All-EFTs-346fb568996b4488afb3252ad5984c50?pvs=21) step and the next `{eft}` in the loop

### Macro 10.2 - Update Records for Data Capture

At the end of each run we should have the entire run object

### Macro 10.3 - Send Run Emails

<aside>
💡 *Note: Use these lists to prepare the email body*

[`run.missing_crossover`] = Crossover Payors Tab

[`run.missing_desc1`] = Waystar Tab

[`run.missing_wspayors`] = WS Payers Tab

</aside>

1. Email Header
    1. to: [marissa.mayfield@allmh.com](mailto:marissa.mayfield@allmh.com)
    2. cc: [kathrynne.johns@allmh.com](mailto:kathrynne.johns@allmh.com)
    3. from: [thoughtful@allmh.com](mailto:thoughtful@allmh.com)
2. Prepare Email Body
    
    Please review and correct/add the following missing items in the AMH2 Mapping.xlsx file [HERE](https://allegianceambulance-my.sharepoint.com/:x:/r/personal/thoughtful_allmh_com/_layouts/15/Doc.aspx?sourcedoc=%7BCBCE1B8B-6912-4141-A453-437B3E6737F9%7D&file=AMH2%20Mapping.xlsx&action=default&mobileredirect=true).
    
    **Crossover Payors Tab**
    
    - payer1
    - payer2
    - payer3
        
        …
        
    - payer?
    
    **WS Payers Tab**
    
    - payer1
    - payer2
    - payer3
        
        …
        
    - payer?
    
    **Waystar Tab**
    
    - payer1
    - payer2
    - payer3
        
        …
        
    - payer?

### Macro 10.4 - Update Run Report

1. For each `{eft}` in `{run.efts}`:
    1. Is `{eft.posted}` = “”?
        1. Y - Update `{eft.posted}` = “P”
        2. N - Go to the next step
    2. For each `{txn}` in `{eft.txns}`:
        1. For each `{change}` in `{eft.changes}`:
            - Go to the next step
2. Update the Run Report
    1. `RUN DATE` = `{run.date}`
    2. `POST DATE` = `{run.post_date}`
    3. `PAYOR` = `{eft.payor}`
    4. `CHECK NUM` = `{eft.check_num}`
    5. `DEPOSIT DATE` = `{eft.txn_date}`
    6. `AMOUNT` = `{eft.amount}`
    7. `POSTED` = `{eft.posted}`
    8. `NOTES` = `{eft.notes}`
    9. `TXN ID` = `{txn.id}`
    10. `LEG ID` = `{txn.leg_id}`
    11. `CHANGE TYPE` = `{change.type}`
    12. `CHANGE AMT` = `{change.amount}`
    13. `CHANGE NOTE` = `{change.note}`
    
    | RUN DATE | POST DATE | PAYOR | CHECK NUM | DEPOSIT DATE | AMOUNT | POSTED | NOTES | TXN ID | LEG ID | CHANGE TYPE | CHANGE AMT | CHANGE NOTE |
    | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
    |  |  |  |  |  |  |  |  |  |  |  |  |  |
    |  |  |  |  |  |  |  |  |  |  |  |  |  |
3. Sort by `TXN ID`

# III. Technical Documentation

## Empower Setup

*Note: Make sure that `{run.date}` is hardcoded to Eastern Time Zone*

### **Empower Selections**

[Macro 1.1 - Get Selections from Empower](https://www.notion.so/Macro-1-1-Get-Selections-from-Empower-8d0f2f81cacb4fa68db8b5e8604739d8?pvs=21) 

### **Empower Run Statuses**

- **Success**:
- **Warning**:
- **Failure**:

### **Empower Artifacts**

- Run Report =
- Register Report downloaded from Cadence Bank

## Data Structure

| OBJECT | REFERENCE | DATA TYPE | NOTES |
| --- | --- | --- | --- |
| {run} |  |  |  |
|  | {run.date} | date | This is the date the Bot runs from Empower |
|  | {run.post_date} | date | May not use this |
|  | [run.ws_payors] | list[str] | List of Waystar Payors from Empower Selections |
|  | [run.missing_desc1] | list[str] | List of missing descriptions |
|  | [run.missing_crossover] | list[str] | List of missing crossover payors |
|  | [run.missing_wspayors] | list[str] | List of missing Waystar payors |
|  | [run.efts] | list[eft] | Filtered List of efts to run from Bank Deposits |
| {eft} |  |  |  |
|  | {eft.check_num} | str |  |
|  | {eft.txn_date} | date |  |
|  | {eft.amount} | float |  |
|  | {eft.desc1} | str |  |
|  | {eft.desc2} | str |  |
|  | {eft.desc3} | str |  |
|  | {eft.835} | str |  |
|  | {eft.waystar_id} | str |  |
|  | {eft.matched} | str |  |
|  | {eft.835_file} | str |  |
|  | {eft.posted} | str |  |
|  | {eft.notes} | str |  |
|  | {eft.empower} | str |  |
|  | {eft.payor} | str |  |
|  | [eft.providers] | list[str] | List of Provider Payment Rows |
|  | [eft.txns] | list[txn] | List of EFT Transactions |
| {txn} |  |  |  |
|  | {txn.id} | str |  |
|  | {txn.leg_id} | str |  |
|  | {txn.status} | str |  |
|  | {txn.crossover} | str |  |
|  | {txn.dos} | date |  |
|  | [txn.changes] | list[change] |  |
| {change} |  |  |  |
|  | {change.type} | str |  |
|  | {change.amount} | float |  |
|  | {change.note} | str |  |

## Run Report

| RUN DATE | POST DATE | PAYOR | CHECK NUM | DEPOSIT DATE | AMOUNT | POSTED | NOTES | TXN ID | LEG ID | CHANGE TYPE | CHANGE AMT | CHANGE NOTE |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
|  |  |  |  |  |  |  |  |  |  |  |  |  |
|  |  |  |  |  |  |  |  |  |  |  |  |  |

# IV. Test Plan

## KickOff

- [ ]  MAP Scope Split?
    
    not applicable for this one
    
- [ ]  QA Purple Color in the MAP
    
    Indicates QA flow. Should be a call out
    
- [x]  Jira board
    
    Do we have it? @Sergey Vystavkin owns this
    
- [x]  T - Bug Catcher
    
    Will we use it? @Sergey Vystavkin owns this
    
- [x]  INPUTS.USE_TEST_DATA mode
    
    Purple Color flow. Controlled from Empower DEV. @Bohdan Sukhov 
    
    5 days window for test data @Jodi Silverman to make sure
    
- [x]  Code Coverage
    
    To set up. @Bohdan Sukhov 
    
- [x]  MAP DAta Types
    
    @Sergey Vystavkin to add
    
- [x]  Report Format
    
    Do we have a Notes section where bot would put decision made per row?
    CHANGE TYPE, CHANGE AMT, CHANGE NOTE - achieve that goal
    
- [x]  Test Environment
    
    Do we have it. Provide an info in appropriate section
    
    We dont have it. Use LIVE
    
- [x]  Test Levels
    
    Notify everyone
    
- [x]  Test Data Set
    
    Notify everyone. @Bohdan Sukhov owns this. Once we create Test Case we do another call where we discuss how do we push the bot to process the data required for the test cases
    
- [x]  Would it be possible to create entities?
    
    Get an answer. Also discussion when we have TCs in place
    
- [x]  MFTD
    
    Present to everyone @Sergey Vystavkin owns this
    

## Test Environment

We utilize the LIVE Environment. It's safe to add, edit, or delete only our records. Do not alter the client's records.

Ensure that we don't import records that the client has already imported

After we add the record - ensure that we delete it in the end of the month (@Jodi Silverman owns that removal)

## Test Levels

| Type | Tracking Sheet | Created By | Performed By | Sign Off |
| --- | --- | --- | --- | --- |
| System Testing | Untitled (https://www.notion.so/0d45828726384506bd914891039785b1?pvs=21)  | LIE/CX | IE/LIE/CX |  |
| User Acceptance Testing | CX/Client would be involved in the System Testing | CX | Client/CX |  |

## Test Data Set

| id | Object | Test Case |
| --- | --- | --- |
| 1 | {} | TBA |

## **Manual Functional Testing Document (MFTD)**

|  | Acceptance Criteria | link to empower |  |  |  |
| --- | --- | --- | --- | --- | --- |
| Empower Execution Date | not empty |  |  |  |  |
| Payors | at least one |  |  |  |  |
| Options | at least one |  |  |  |  |
| Data Source | LIVE_DATA / TEST_DATA |  |  |  |  |
| DEV_SAFE_MODE | True / False |  |  |  |  |
| TARGET_ENTITIES | [] |  |  |  |  |
| Executed By | not empty |  |  |  |  |
| Empower Environment | DEV / PRD |  |  |  |  |
| Automation Environment | DEV / PRD |  |  |  |  |
| Empower Run Result | success |  |  |  |  |
| Empower Execution Duration | < 2 hours |  |  |  |  |
| Successfully Processed Entities | > 0 |  |  |  |  |
| Entities Processed with Errors | == 0 |  |  |  |  |
| t-bugcatcher Errors | == 0 |  |  |  |  |
| t-bugcatcher Errors | == 0 |  |  |  |  |
| t-bugcatcher Errors | == 0 |  |  |  |  |
| Total Code Coverage, % | > 70 |  |  |  |  |
| Test Case 1 | Code Coverage + 
Message in Report + 
Message in Logs + 
Manually verified in the App |  |  |  |  |
| Test Case … | Code Coverage + 
Message in Report + 
Message in Logs + 
Manually verified in the App |  |  |  |  |
| Test Case X | Code Coverage + 
Message in Report + 
Message in Logs + 
Manually verified in the App |  |  |  |  |
| Comment | not empty |  |  |  |  |
| Approved By | @Sergey Vystavkin @Jodi Silverman  |  |  |  |  |