# leukemia-database
Python tools used for connecting to and manipulating data stored in REDCap, SQLServer, and MySQL

# Read library files


```python
# import pymysql as db  # comes with utilities
# import pandas as pd   # comes with utilities
# dependenct on config_.ini setup and utility directory location

import os,sys,inspect   # sys is part of the utilities, retained here so we can access directory pathing

# set up sub path directories
# replace parent/gramps with following!  :)
current_dir = os.path.dirname(os.path.abspath(inspect.getfile(inspect.currentframe())))
utility_dir = current_dir
config_dir  = current_dir
onedrv_dir  = "C:/Users/cmshaw/OneDrive - Fred Hutchinson Cancer Research Center"

# utility path
while not os.path.exists(utility_dir + '\\Utility'):
    utility_dir = os.path.dirname(utility_dir)
utility_dir = utility_dir + '\\Utility'  
sys.path.insert(0, utility_dir)

# configuration path
while not os.path.exists(config_dir + '\\Config'):
    config_dir = os.path.dirname(config_dir)
config_dir = config_dir + '\\Config'  
sys.path.insert(0, config_dir)
configfile = config_dir +"\\config_.ini"

# import mysql.connector and mysql engine utilities
from utils import *
from redcapengine import *

print("Current Directory:")
print(current_dir)
print("Utility Directory:")
print(utility_dir)
print("Config Directory:")
print(config_dir)
print("One-Drive Directory:")
print(onedrv_dir)
print()
print(sys.version)

# everytime this program runs a copy is made and stamped with the current hour/min/seconds
timenow   = gettime('%H%M%S')
today     = gettime('%Y%m%d')
todaydash = gettime('%Y-%m-%d')
todaytime = gettime('%Y-%m-%d %H:%M:%S')
print(todaytime)

# handy constants
slash = '\\'
dblslash = '\\\\'

# set up emailer object
Emailer = EmailMessage(sect='hutchemail')

# set up "Do" vars
DoMe    = True
SkipMe  = False
DoAll   = True
SkipAll = False
DoDebug = False # or DoDebug = True
```

    running
    # Not testing engine load
    # testing turned off
    Current Directory:
    C:\Users\cmshaw\WorkingNotebooks\Templates
    Utility Directory:
    C:\Users\cmshaw\WorkingNotebooks\Utility
    Config Directory:
    C:\Users\cmshaw\WorkingNotebooks\Config
    One-Drive Directory:
    C:/Users/cmshaw/OneDrive - Fred Hutchinson Cancer Research Center
    
    3.8.5 (default, Sep  3 2020, 21:29:08) [MSC v.1916 64 bit (AMD64)]
    2023-09-22 10:39:00
    {'ini_section': 'hutchemail', 'user': 'cmshaw@fredhutch.org', 'password': 'WarHammer-2022', 'today': '20230922'}
    

# MySQLEngine Class utilizing Data Connection Class

## Setup data connection for MySQL


```python
# Setup data connection for MySQL
# return connection object which contains the actual connection
schema = 'some_mysql_schema'
sect   = 'some_config_section'

conobj     = DataConnection( sect=schema, schema=schema, sqlalchemy=True, feedback='err')
alchemycon = conobj.con

conobj     = DataConnection( sect=schema, schema=schema, sqlalchemy=False, feedback='err')
mysqlcon   = conobj.con

# conobj   = DataConnection( sect='caisisprod', schema=schema, sqlalchemy=False, feedback='err')
# sqlservcon = conobj.con
sqlservcon = None

print(alchemycon, mysqlcon, sqlservcon)

# create this subclassed sql engine object (mysql or odbc for sql server) for Arrival functions
# this is the 'standard' data processing engine
mysqleng = MySQLEngine(schema=schema)
mysqleng.addconnection(mysqlcon)
mysqleng.addconnection(alchemycon)       
mysqleng.addconnection(sqlservcon)
mysqleng.con = mysqleng.mysqlcon

mysqleng.feedback='err'

# check engine connectivity, returns boolean
isactive = mysqleng.checkeng()
print(f'\r\nEngine active: {isactive}')

# Object Properties
# in this example properties of a mysql engine class
mysqleng.__dict__
```

# REDCap Engine Class utilizing Data Connection Class

## Set up database connections and engine for REDCap data

If you are connecting to multiple REDCap projects, you need to add a suffix to the object name.  So rather than 'redcapeng' your engines could be 'redcapeng1' and 'redcapeng2', can't have two objects with the same name!!


```python
# Set up database connections and engine for REDCap data

# the redcap dictionary enhances the regular mysql dictionary to add on 

# Creating a data connection requires a conguration definition in config_.ini
# Here we have two connection objects, 1) for running sql against mysql, 2) for running sql against SQLserver
schema = 'some_redcap_project'
sect   = 'some_config_section'
conobj = DataConnection( sect=sect, schema=schema, sqlalchemy=False, feedback='err')
mysqlcon  = conobj.con
mycondict = conobj.condict

conobj = DataConnection( sect=sect, schema=schema, sqlalchemy=True, feedback='err')
alchemycon = conobj.con

# Creating a REDCap engine requires a connection dictionary
redcapeng = REDCapEngine(condict=mycondict,feedback='err')
redcapeng.addconnection(con=alchemycon)
redcapeng.addconnection(con=mysqlcon)
# builds the keytable
redcapeng.rc_key()
'Done'
```

# REDCap SchemaDictionary Class
## Create dictionary, coding table and backups
run occasionally when data needs refreshed.

requires a connection dictionary and a redcap engine, optionally you can specify the connection object to use.

### required parameters:

- condict: a connection dictionary from the DataConnection class
    - i.e. mycondict = DataConnection( sect=sect, schema=schema, sqlalchemy=False, feedback='err').condict

- engine: an engine for getting data from a redcap project replicated to MySQL from the RedCapEngine class
    - i.e. redcapeng = REDCapEngine(condict=mycondict,feedback='err')

### optional parameters:
- con:  a connection object.  Defaults to the engine's con object
    - i.e. con = REDCapEngine(condict=mycondict,feedback='err').con
- recreate:  True (default)/False 
    - causes the schema dictionary to be created, false suppresses creation/changes
- codedict: True/False (default)
    - causes the code dictionary to be created, false suppresses creation/changes
- tbl: the name of a form or table in the RedCap project
    - causes only the specific form/table coding to be updated (exactly the same as setting frm parameter)
- frm: the name of a form or table in the RedCap project
    - causes only the specific form/table coding to be updated (exactly the same as setting tbl parameter)




```python
# Create dictionary, coding table and backups
# note that Schema building requires a mysqleng and a data dictionary from a REDCap connection
redcapeng.feedback='err'
schema = 'some_redcap_project'
redcapdict = SchemaDictionary(
    con=redcapeng.mysqlcon
    , condict=mycondict
    , engine=redcapeng
    , schema=schema
    , codedict=True
    , recreate=True
    # Specific table to update coding for, defaults to all tables 
    , tbl='sometablename' 
    # , frm='someformname' can be used alternatively instead of tbl
    )
```


```python
# uploads all the REDCap instruments into the MySQL schema
forms_df = redcapeng.rc_forms(suppress='obsolete_fields')
```

## Additional functionality

This class is used to create sql case statements useful when querying coded REDCap variables

For example you can return a case statement that will convert a variable coded as an integer value into the meaning associated with that value as found in the REDCap source project

### sample radio or dropdown variable code conversion

```sql
-- integer to meaning, when the codes are integer
CASE
    WHEN `arrivaltype1`=1  THEN "Newly Diagnosed [1]"
    WHEN `arrivaltype1`=2  THEN "Relapse Salvage [2]"
    WHEN `arrivaltype1`=3  THEN "Refractory Salvage [3]"
    WHEN `arrivaltype1`=4  THEN "MRD Relapse [4]"
    WHEN `arrivaltype1`=5  THEN "MRD Refractory [5]"
    WHEN `arrivaltype1`=8  THEN "Dx Unknown [8]"
    WHEN `arrivaltype1`=9  THEN "Other [9]"
    ELSE NULL
END AS `arrivaltype1_meaning`
```
```sql
--  text to meaning, when the codes are text
CASE
    WHEN `rxline1`="-1" THEN "Not entered [-1]"
    WHEN `rxline1`="HCT" THEN "Transplant"
    WHEN `rxline1`="induct" THEN "Ind"
    WHEN `rxline1`="S1" THEN "S1"
    WHEN `rxline1`="S2" THEN "S2"
    WHEN `rxline1`="S3" THEN "S3"
    WHEN `rxline1`="S4" THEN "S4"
    WHEN `rxline1`="S5" THEN "S5"
    WHEN `rxline1`="S6" THEN "S6"
    WHEN `rxline1`="S7" THEN "S7"
    WHEN `rxline1`="S8" THEN "S8"
    WHEN `rxline1`="S9" THEN "S9"
    WHEN `rxline1`="S10" THEN "S10"
    WHEN `rxline1`="S11" THEN "S11"
    ELSE NULL
END AS `rxline1_meaning`
```
```sql
-- meaning to integer code value
CASE
    WHEN `diseaseatarrival`="Newly diagnosed" THEN 1
    WHEN `diseaseatarrival`="Relapsed (at least one previous CR)" THEN 2
    WHEN `diseaseatarrival`="Refractory (no previous CR)" THEN 3
    WHEN `diseaseatarrival`="In remission" THEN 4
    WHEN `diseaseatarrival`="In remission with persistant MRD" THEN 5
    WHEN `diseaseatarrival`="In MRD only relapse" THEN 6
    ELSE NULL
END AS `diseaseatarrival_order`
```
### sample checkbox conversion to meaning
```sql
-- Checkbox controls download from REDCap as multiple variables, one per checkbox option,
-- each variable contains the variable name with a coded suffix.
-- This statement combines the results into a list of the items checked.

-- returns list containing meaning of all checked values
SUBSTR(CONCAT(
    IF(`arrivalreason1___10`, ",>20% [10]", ""),
    IF(`arrivalreason1___11`, ",>10% [11]", ""),
    IF(`arrivalreason1___12`, ",>5% (R/R) [12]", ""),
    IF(`arrivalreason1___13`, ",< 5% (MRD) [13]", ""),
    IF(`arrivalreason1___14`, ",t(15;17) (APL) [14]", ""),
    IF(`arrivalreason1___15`, ",t(8;21) [15]", ""),
    IF(`arrivalreason1___16`, ",inv(16) [16]", ""),
    IF(`arrivalreason1___17`, ",EMD [17]", ""),
    IF(`arrivalreason1___26`, ",AUL [26]", ""),
    IF(`arrivalreason1___25`, ",ABL [25]", ""),
    IF(`arrivalreason1___18`, ",MPAL [18]", ""),
    IF(`arrivalreason1___19`, ",tp53 [19]", ""),
    IF(`arrivalreason1___20`, ",BPDCN [20]", ""),
    IF(`arrivalreason1___31`, ",In CR for Consolidation [31]", ""),
    IF(`arrivalreason1___27`, ",In CR for HCT [27]", ""),
    IF(`arrivalreason1___28`, ",In CR for consult [28]", ""),
    IF(`arrivalreason1___29`, ",In CR w/o MRD for HCT [29]", ""),
    IF(`arrivalreason1___30`, ",In CR w/MRD for HCT [30]", ""),
    IF(`arrivalreason1___32`, ",Other - please specify", ""),
    IF(`arrivalreason1___99`, ",Other WHO AML [99]", "")),2)
```
```sql
-- returns list containing all checked codes
SUBSTR(CONCAT(
    IF(`diagnosis___1`, ",1", ""),
    IF(`diagnosis___2`, ",2", ""),
    IF(`diagnosis___3`, ",3", ""),
    IF(`diagnosis___4`, ",4", ""),
    IF(`diagnosis___5`, ",5", ""),
    IF(`diagnosis___6`, ",6", ""),
    IF(`diagnosis___8`, ",8", ""),
    IF(`diagnosis___7`, ",7", "")),2)
```



```python
# create code dictionary class and the table redcapdictionaryconv
# requires that a code table was created previously
myRCDict = RCCodeDict(eng=mysqleng,schema=schema)
myRCDict.cd['multi'].head(1)
```

# Query object/class definitions

## return methods a.k.a. functions

## return attributes a.k.a. properties


```python
# methods by default does not include double underscore
meth_list = get_methods(some_class_or_obj)
# methods including double underscore
meth_list = get_methods(some_class_or_obj,dunder=True))
# attributes by default does not include double underscore
attrib_list = get_attributes(some_class_or_obj)
# attributes including double underscore
attrib_list = get_attributes(some_class_or_obj,True)
# built in __dir__() function returns both methods and attributes including double underscore
attrib_list = some_class_or_obj.__dir__()
```

# Query data dictionary in REDCap to describe fields

# Upload REDCap instruments to MySQL

## Refresh of the MySQL copy of the REDCap project

# uploads all the REDCap instruments into the MySQL schema
forms_df = redcapeng.rc_forms(suppress='obsolete_fields')
df=redcapeng.rc_key()
df.describe()
# Update specific forms in REDCap

## Refresh specific forms


```python
# Update specific forms in REDCap
df = redcapeng.rc_form(tbl='patient_list')
df = redcapeng.rc_form(tbl='arrival1')
df = redcapeng.rc_form(tbl='kd2008_newlydiagnosed')
df = redcapeng.rc_form(tbl='kd2018_newlydiagnosed')
df = redcapeng.rc_form(tbl='redcap_dx_induction')
```

# Copy WorkDBProd data to MySQL
# Copies tables from WorkDBProd to the MySQL database caisis
mylist = [
    'vdatasetAMLLabs2',
]
MoveEng = SQLMoveEngine(tbllist=mylist, configfile=configfile)
# Send Email from python
# Send email from python 
Emailer = EmailMessage(sect='smtpemail')
msgsubj = 'Message subject'
msgbody = 'Message text'
# comma delimited list of target email addresses
trgemail = r'cmshaw@fredhutch.org, carolemarieshawjunk@gmail.com'
# comma delimited list of files to attach
file     = r'data\image1.jfif,data\image2.jfif'
Emailer.send_email( trgemail
              , msgsubj
              , msgbody
              , file )
# Import Strategies

## Import Specific Excel Tab


```python
df = pd.read_excel('data\\ExcelFile.xlsx', sheet_name='Sheet2')            
mysqleng.tosql(con=mysqleng.alchemycon, df=df, tbl='exceldata', schema='temp')
df
```

## Import CSV data


```python
df = pd.read_csv('data\\CSVFile.csv')            
mysqleng.tosql(con=mysqleng.alchemycon, df=df, tbl='csvdata', schema='temp')
df
```

## Import REDCap Report (CSV)
# grabs data to dataframe and incidentally outputs data to csv
reportdf = redcapeng.rc_report(rpt=1234,csv=r'output\CSVFile.csv')
reportdf
## Import REDCap Report (MySQL Table)


```python
# grabs data to dataframe and incidentally outputs data to mysql table
reportdf = redcapeng.rc_report(rpt=15093,tbl="playground_mrn_list",schema="junk")
reportdf
```

# Export Strategies for Data Frames

## Export dataframe to Excel


```python
# Export dataframe to Excel
df.to_excel(r"output\XLSXFile.xlsx")
```

## Export a dataframe to a CSV


```python
# Export a dataframe to a CSV
df.to_csv(r"output\playground_mrn_arrival_id_list.csv", index=False)
```

## Export a dataframe to a TXT file


```python
# Export a dataframe to a TXT file
df.to_csv(r"output\TextFile.txt", index=False, sep='\t')
```

## Export a dataframe to MySQL


```python
# Export a dataframe to MySQL
rslt = mysqleng.tosql(con=mysqleng.alchemycon, df=df, tbl='sometable', schema='temp')
'Done'
```

# Converting Excel to CSV utilizing csv_from_excel procedure

Useful if your excel file contains karyotype data or other data that contains many delimiting symbols/characters can mess up the way dataframes work

## Excel to CSV defaulting to first tab


```python
# example parameters
dir_   = r'C:\Users\cmshaw\OneDrive - Fred Hutchinson Cancer Research Center\AML Database Resources\EPIC Downloads'
excel_ = f'{today}_UW_CYTO_no_pass.xlsx'   

# default to first tab
mycsv   = csv_from_excel(myexcel=excel_, mydir=dir_)
```

## Excel to CSV by sheet name


```python
# example parameters
dir_   = r'C:\Users\cmshaw\OneDrive - Fred Hutchinson Cancer Research Center\AML Database Resources\EPIC Downloads'
excel_ = f'{today}_UW_CYTO_no_pass.xlsx'   
sheet_ = 'SecondSheet'

# by sheet name
mycsv   = csv_from_excel(myexcel=excel_, mydir=dir_, mysheet=sheet_)
```

## Excel to CSV by sheet position


```python
# example parameters
dir_   = r'C:\Users\cmshaw\OneDrive - Fred Hutchinson Cancer Research Center\AML Database Resources\EPIC Downloads'
excel_ = f'{today}_UW_CYTO_no_pass.xlsx'   

# by sheet position, this is zero based so the first sheet is sheet 0
mycsv   = csv_from_excel(myexcel=excel_, mydir=dir_, mysheet=0)
```

# Managing Excel Output


```python
if __name__ == '__main__':
    data_first = {'column_a': pd.Series(data=np.random.randint(10, 100, 3),
                                        index=['a', 'b', 'c']),
                  'column_b': pd.Series(data=np.random.randint(10, 100, 3),
                                        index=['a', 'b', 'c']),
                  'column_c': pd.Series(data=np.random.randint(10, 100, 3),
                                        index=['a', 'b', 'c'])}
    data_second = {'column_d': pd.Series(data=np.random.randint(10, 100, 3),
                                         index=['d', 'e', 'f']),
                   'column_e': pd.Series(data=np.random.randint(10, 100, 3),
                                         index=['d', 'e', 'f']),
                   'column_f': pd.Series(data=np.random.randint(10, 100, 3),
                                         index=['d', 'e', 'f'])}
    df1 = pd.DataFrame(data=data_first)
    df2 = pd.DataFrame(data=data_second)
```

## Export dataframe to an excel file


```python
# Export dataframe to an excel file
df1.to_excel(r"output\XLSXFile.xlsx")
```

## Two dataframes to a single excel tab


```python
# Two dataframes on one tab
df1 = df
df2 = df
# starts 2 rows after the first frame
with pd.ExcelWriter(r'output\two_frames_one_tab.xlsx', engine='xlsxwriter') as writer:
    df1.to_excel(writer, sheet_name='data', index=False)
    df2.to_excel(writer, sheet_name='data', index=False,
                       startrow=(df1.shape[0] + 3))
```

## Two dataframes side-by-side to a single excel tab


```python
# Two dataframes side-by-side
df1 = df
df2 = df
# starts 2 rows after the first frame
with pd.ExcelWriter(r'output\two_frames_side_by_side.xlsx', engine='xlsxwriter') as writer:
    df1.to_excel(writer, sheet_name='data', index=False)
    df2.to_excel(writer, sheet_name='data', index=False,
                       startcol=(df1.shape[1] + 1))
```

## Multiple tabs in one excel workbook on seperate tabs


```python
# Multiple tabs in one workbook
df1 = df
df2 = df
with pd.ExcelWriter(r'output\multiple_tabs.xlsx', engine='xlsxwriter') as writer:
    df1.to_excel(writer, sheet_name='first_tab', index=False)
    df2.to_excel(writer, sheet_name='second_tab', index=False)        
```

## Auto adjust excel column

### Fix column widths of excel file


```python
# Fix column widths of excel file
fix_col_width(srcfile='data\\ExcelFile.xlsx')
```

### Fix column widths of excel file, output to new excel


```python
# Fix column widths of excel file, output to new excel
fix_col_width(srcfile='data\\ExcelFile.xlsx', trgfile='output\ExcelColWidths.xlsx', sheet='Sheet1')
```

### Export Excel to an Image


```python
# import excel2img
# excel2img.export_img(fn_excel, fn_image, page=None, _range=None)
excel2img.export_img(fn_excel='data\\ExcelFile.xlsx'
                     , fn_image="output\\ExcelFile1.bmp"
                     , page="Sheet1")
excel2img.export_img(fn_excel='data\\ExcelFile.xlsx'
                     , fn_image="output\\ExcelFile2.bmp"
                     , _range="Sheet1!A1:H10")
excel2img.export_img(fn_excel='data\\ExcelFile.xlsx'
                     , fn_image="output\\ExcelFile3.bmp"
                     , _range="Sheet1!range1")
```

# Iterate files in a directory


```python
# iterate files in directory import os
directory = os.fsencode(directory_in_str)
    
for file in os.listdir(directory):
    filename = os.fsdecode(file)
    if filename.endswith(".asm") or filename.endswith(".py"): 
        # print(os.path.join(directory, filename))
        continue
    else:
        continue
```

# Query data dictionary in REDCap to describe fields


```python
# Query data dictionary in REDCap to describe fields
col_list = tuple(df.columns)
schema   = 'SomeSchema'
cmd = f"""
    SELECT field_name, field_label, select_choices_or_calculations 
        FROM `{schema}`.`redcapdictionary` 
        WHERE LOCATE(field_name,"{col_list}")>0 ;
"""
rslt = mysqleng.dosql(cmd=cmd,con=mysqleng.mysqlcon)
print(cmd)
'Done'
```

# REDCap coding class


```python
# returns a class object that knows about how to work with REDCap coding
rccode = REDCapCoding(engine=mysqleng)
```

## dataframe of field coding


```python
# returns a dataframe of the coding for the passed in field
df = rccode.rc_fieldcode(fld='arrivalsubchar1')
df
```

## string of REDCap dictionary choices


```python
# returns a string of the choices as defined in the REDCap dictionary
mychoices = rccode.rc_choices('arrivalsubchar1')
mychoices
```

# Get REDCap Report Structure

Useful if you want to move a report's data to another project


```python
# returns a class object that knows how to work with REDCap coding
rccodeeng = REDCapCoding(engine=mysqleng,rceng=redcapeng)
# returns a dataframe based on the report that contains the structure of the source project's fields
# incidentally outputs a file in the output directory with the dictionary structure
# also replicates the structure to a temp table in the temp schema
# passing in a tbl will update the resulting structure to place all fields in that form(instrument)
df = rccodeeng.rc_reportdict(reportid=238819,tbl='form_1')
```

# Formatting DataFrame Output

## Pretty Print for DataFrames

Control width of columns, and fieldlist.

Print multiple dataframes prettily as output from a jupyter notebook cell.

```python
syntax:    ppdf(df,width,fieldlist):
```


```python
# create sample dataframe
df = pd.DataFrame([{'ptmrn': 'U1234567', 'var1': 'a,b,c'*1000, 'var2': 1, 'onemorevar':  0.457},
                   {'ptmrn': 'U0000001', 'var1': 'd,e,f'*1000, 'var2': 2, 'onemorevar': -3.815},
                   {'ptmrn': 'U0000002', 'var1': 'g,h,i'*1000, 'var2': 2, 'onemorevar':  3.815},
                    {'ptmrn': 'U0000003', 'var1': 'j,k,l'*1000, 'var2': 2, 'onemorevar': -2.111},
                    {'ptmrn': 'U0000004', 'var1': 'm,n,o'*1000, 'var2': 2, 'onemorevar':  2.111},
                    {'ptmrn': 'U0000005', 'var1': 'p,q,r'*1000, 'var2': 2, 'onemorevar': -1.222},
                    {'ptmrn': 'U0000006', 'var1': 's,t,u'*1000, 'var2': 2, 'onemorevar':  1.222},
                    {'ptmrn': 'U0000007', 'var1': 'v,w,x'*1000, 'var2': 2, 'onemorevar': -3.333},
                    {'ptmrn': 'U0000008', 'var1': 'y,z,1'*1000, 'var2': 2, 'onemorevar':  3.333},
                    {'ptmrn': 'U0000009', 'var1': '1,2,3'*1000, 'var2': 2, 'onemorevar':  4.333},
                   {'ptmrn': 'U0000010', 'var1': '4,5,6'*1000, 'var2': 2, 'onemorevar':  4.444},
                   {'ptmrn': 'U0000011', 'var1': '7,8,9'*1000, 'var2': 2, 'onemorevar': -5.555},
                   {'ptmrn': 'U0000012', 'var1': '0,@,%'*1000, 'var2': 2, 'onemorevar':  5.555}])
```

### pretty print dataframe all fields, limit to the first 25 char for each column


```python
## pretty print dataframe all fields, limit to the first 25 char for each column
ppdf(df, width=25)
```


<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>ptmrn</th>
      <th>var1</th>
      <th>var2</th>
      <th>onemorevar</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>U1234567</td>
      <td>a,b,ca,b,ca,b,ca,b,ca...</td>
      <td>1</td>
      <td>0.457</td>
    </tr>
    <tr>
      <th>1</th>
      <td>U0000001</td>
      <td>d,e,fd,e,fd,e,fd,e,fd...</td>
      <td>2</td>
      <td>-3.815</td>
    </tr>
    <tr>
      <th>2</th>
      <td>U0000002</td>
      <td>g,h,ig,h,ig,h,ig,h,ig...</td>
      <td>2</td>
      <td>3.815</td>
    </tr>
    <tr>
      <th>3</th>
      <td>U0000003</td>
      <td>j,k,lj,k,lj,k,lj,k,lj...</td>
      <td>2</td>
      <td>-2.111</td>
    </tr>
    <tr>
      <th>4</th>
      <td>U0000004</td>
      <td>m,n,om,n,om,n,om,n,om...</td>
      <td>2</td>
      <td>2.111</td>
    </tr>
    <tr>
      <th>5</th>
      <td>U0000005</td>
      <td>p,q,rp,q,rp,q,rp,q,rp...</td>
      <td>2</td>
      <td>-1.222</td>
    </tr>
    <tr>
      <th>6</th>
      <td>U0000006</td>
      <td>s,t,us,t,us,t,us,t,us...</td>
      <td>2</td>
      <td>1.222</td>
    </tr>
    <tr>
      <th>7</th>
      <td>U0000007</td>
      <td>v,w,xv,w,xv,w,xv,w,xv...</td>
      <td>2</td>
      <td>-3.333</td>
    </tr>
    <tr>
      <th>8</th>
      <td>U0000008</td>
      <td>y,z,1y,z,1y,z,1y,z,1y...</td>
      <td>2</td>
      <td>3.333</td>
    </tr>
    <tr>
      <th>9</th>
      <td>U0000009</td>
      <td>1,2,31,2,31,2,31,2,31...</td>
      <td>2</td>
      <td>4.333</td>
    </tr>
    <tr>
      <th>10</th>
      <td>U0000010</td>
      <td>4,5,64,5,64,5,64,5,64...</td>
      <td>2</td>
      <td>4.444</td>
    </tr>
    <tr>
      <th>11</th>
      <td>U0000011</td>
      <td>7,8,97,8,97,8,97,8,97...</td>
      <td>2</td>
      <td>-5.555</td>
    </tr>
    <tr>
      <th>12</th>
      <td>U0000012</td>
      <td>0,@,%0,@,%0,@,%0,@,%0...</td>
      <td>2</td>
      <td>5.555</td>
    </tr>
  </tbody>
</table>
</div>





    True



### pretty print dataframe fieldlist, limit to the first 50 char for each column


```python
## pretty print dataframe fieldlist, limit to the first 50 char for each column
fieldlist=['ptmrn','onemorevar','var1']
ppdf(df,50,fieldlist)
```


<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>ptmrn</th>
      <th>onemorevar</th>
      <th>var1</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>U1234567</td>
      <td>0.457</td>
      <td>a,b,ca,b,ca,b,ca,b,ca,b,ca,b,ca,b,ca,b,ca,b,ca...</td>
    </tr>
    <tr>
      <th>1</th>
      <td>U0000001</td>
      <td>-3.815</td>
      <td>d,e,fd,e,fd,e,fd,e,fd,e,fd,e,fd,e,fd,e,fd,e,fd...</td>
    </tr>
    <tr>
      <th>2</th>
      <td>U0000002</td>
      <td>3.815</td>
      <td>g,h,ig,h,ig,h,ig,h,ig,h,ig,h,ig,h,ig,h,ig,h,ig...</td>
    </tr>
    <tr>
      <th>3</th>
      <td>U0000003</td>
      <td>-2.111</td>
      <td>j,k,lj,k,lj,k,lj,k,lj,k,lj,k,lj,k,lj,k,lj,k,lj...</td>
    </tr>
    <tr>
      <th>4</th>
      <td>U0000004</td>
      <td>2.111</td>
      <td>m,n,om,n,om,n,om,n,om,n,om,n,om,n,om,n,om,n,om...</td>
    </tr>
    <tr>
      <th>5</th>
      <td>U0000005</td>
      <td>-1.222</td>
      <td>p,q,rp,q,rp,q,rp,q,rp,q,rp,q,rp,q,rp,q,rp,q,rp...</td>
    </tr>
    <tr>
      <th>6</th>
      <td>U0000006</td>
      <td>1.222</td>
      <td>s,t,us,t,us,t,us,t,us,t,us,t,us,t,us,t,us,t,us...</td>
    </tr>
    <tr>
      <th>7</th>
      <td>U0000007</td>
      <td>-3.333</td>
      <td>v,w,xv,w,xv,w,xv,w,xv,w,xv,w,xv,w,xv,w,xv,w,xv...</td>
    </tr>
    <tr>
      <th>8</th>
      <td>U0000008</td>
      <td>3.333</td>
      <td>y,z,1y,z,1y,z,1y,z,1y,z,1y,z,1y,z,1y,z,1y,z,1y...</td>
    </tr>
    <tr>
      <th>9</th>
      <td>U0000009</td>
      <td>4.333</td>
      <td>1,2,31,2,31,2,31,2,31,2,31,2,31,2,31,2,31,2,31...</td>
    </tr>
    <tr>
      <th>10</th>
      <td>U0000010</td>
      <td>4.444</td>
      <td>4,5,64,5,64,5,64,5,64,5,64,5,64,5,64,5,64,5,64...</td>
    </tr>
    <tr>
      <th>11</th>
      <td>U0000011</td>
      <td>-5.555</td>
      <td>7,8,97,8,97,8,97,8,97,8,97,8,97,8,97,8,97,8,97...</td>
    </tr>
    <tr>
      <th>12</th>
      <td>U0000012</td>
      <td>5.555</td>
      <td>0,@,%0,@,%0,@,%0,@,%0,@,%0,@,%0,@,%0,@,%0,@,%0...</td>
    </tr>
  </tbody>
</table>
</div>





    True



### pretty print dataframe fieldlist, just the header


```python
## pretty print dataframe fieldlist, just the header
fieldlist=['ptmrn','onemorevar','var1']
ppdf(df.head(),50,fieldlist)
```


<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>ptmrn</th>
      <th>onemorevar</th>
      <th>var1</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>U1234567</td>
      <td>0.457</td>
      <td>a,b,ca,b,ca,b,ca,b,ca,b,ca,b,ca,b,ca,b,ca,b,ca...</td>
    </tr>
    <tr>
      <th>1</th>
      <td>U0000001</td>
      <td>-3.815</td>
      <td>d,e,fd,e,fd,e,fd,e,fd,e,fd,e,fd,e,fd,e,fd,e,fd...</td>
    </tr>
    <tr>
      <th>2</th>
      <td>U0000002</td>
      <td>3.815</td>
      <td>g,h,ig,h,ig,h,ig,h,ig,h,ig,h,ig,h,ig,h,ig,h,ig...</td>
    </tr>
    <tr>
      <th>3</th>
      <td>U0000003</td>
      <td>-2.111</td>
      <td>j,k,lj,k,lj,k,lj,k,lj,k,lj,k,lj,k,lj,k,lj,k,lj...</td>
    </tr>
    <tr>
      <th>4</th>
      <td>U0000004</td>
      <td>2.111</td>
      <td>m,n,om,n,om,n,om,n,om,n,om,n,om,n,om,n,om,n,om...</td>
    </tr>
  </tbody>
</table>
</div>





    True



### pretty print dataframe, just the first 8 header rows


```python
## pretty print dataframe, just the first 8 header rows
ppdf(df.head(8),40)
```


<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>ptmrn</th>
      <th>var1</th>
      <th>var2</th>
      <th>onemorevar</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>U1234567</td>
      <td>a,b,ca,b,ca,b,ca,b,ca,b,ca,b,ca,b,ca...</td>
      <td>1</td>
      <td>0.457</td>
    </tr>
    <tr>
      <th>1</th>
      <td>U0000001</td>
      <td>d,e,fd,e,fd,e,fd,e,fd,e,fd,e,fd,e,fd...</td>
      <td>2</td>
      <td>-3.815</td>
    </tr>
    <tr>
      <th>2</th>
      <td>U0000002</td>
      <td>g,h,ig,h,ig,h,ig,h,ig,h,ig,h,ig,h,ig...</td>
      <td>2</td>
      <td>3.815</td>
    </tr>
    <tr>
      <th>3</th>
      <td>U0000003</td>
      <td>j,k,lj,k,lj,k,lj,k,lj,k,lj,k,lj,k,lj...</td>
      <td>2</td>
      <td>-2.111</td>
    </tr>
    <tr>
      <th>4</th>
      <td>U0000004</td>
      <td>m,n,om,n,om,n,om,n,om,n,om,n,om,n,om...</td>
      <td>2</td>
      <td>2.111</td>
    </tr>
    <tr>
      <th>5</th>
      <td>U0000005</td>
      <td>p,q,rp,q,rp,q,rp,q,rp,q,rp,q,rp,q,rp...</td>
      <td>2</td>
      <td>-1.222</td>
    </tr>
    <tr>
      <th>6</th>
      <td>U0000006</td>
      <td>s,t,us,t,us,t,us,t,us,t,us,t,us,t,us...</td>
      <td>2</td>
      <td>1.222</td>
    </tr>
    <tr>
      <th>7</th>
      <td>U0000007</td>
      <td>v,w,xv,w,xv,w,xv,w,xv,w,xv,w,xv,w,xv...</td>
      <td>2</td>
      <td>-3.333</td>
    </tr>
  </tbody>
</table>
</div>





    True



## Tabulate output for dataframe


```python
# Tabulate output for dataframe
print(tabulate(df[['ptmrn','var2','onemorevar']], headers = 'keys', tablefmt = 'psql'))
```

    +----+----------+--------+--------------+
    |    | ptmrn    |   var2 |   onemorevar |
    |----+----------+--------+--------------|
    |  0 | U1234567 |      1 |        0.457 |
    |  1 | U0000001 |      2 |       -3.815 |
    |  2 | U0000002 |      2 |        3.815 |
    |  3 | U0000003 |      2 |       -2.111 |
    |  4 | U0000004 |      2 |        2.111 |
    |  5 | U0000005 |      2 |       -1.222 |
    |  6 | U0000006 |      2 |        1.222 |
    |  7 | U0000007 |      2 |       -3.333 |
    |  8 | U0000008 |      2 |        3.333 |
    |  9 | U0000009 |      2 |        4.333 |
    | 10 | U0000010 |      2 |        4.444 |
    | 11 | U0000011 |      2 |       -5.555 |
    | 12 | U0000012 |      2 |        5.555 |
    +----+----------+--------+--------------+
    


```python
# styling with color
def color_negative_red(val):
    """
    Takes a scalar and returns a string with
    the css property `'color: red'` for negative
    strings, black otherwise.
    """
    color = 'red' if val < 0 else 'black'
    return 'color: % s' % color
  
# displaying the DataFrame
df[['var2','onemorevar']].style.applymap(color_negative_red)
```




<style  type="text/css" >
#T_ef86a5cb_596e_11ee_b355_548d5a571682row0_col0,#T_ef86a5cb_596e_11ee_b355_548d5a571682row0_col1,#T_ef86a5cb_596e_11ee_b355_548d5a571682row1_col0,#T_ef86a5cb_596e_11ee_b355_548d5a571682row2_col0,#T_ef86a5cb_596e_11ee_b355_548d5a571682row2_col1,#T_ef86a5cb_596e_11ee_b355_548d5a571682row3_col0,#T_ef86a5cb_596e_11ee_b355_548d5a571682row4_col0,#T_ef86a5cb_596e_11ee_b355_548d5a571682row4_col1,#T_ef86a5cb_596e_11ee_b355_548d5a571682row5_col0,#T_ef86a5cb_596e_11ee_b355_548d5a571682row6_col0,#T_ef86a5cb_596e_11ee_b355_548d5a571682row6_col1,#T_ef86a5cb_596e_11ee_b355_548d5a571682row7_col0,#T_ef86a5cb_596e_11ee_b355_548d5a571682row8_col0,#T_ef86a5cb_596e_11ee_b355_548d5a571682row8_col1,#T_ef86a5cb_596e_11ee_b355_548d5a571682row9_col0,#T_ef86a5cb_596e_11ee_b355_548d5a571682row9_col1,#T_ef86a5cb_596e_11ee_b355_548d5a571682row10_col0,#T_ef86a5cb_596e_11ee_b355_548d5a571682row10_col1,#T_ef86a5cb_596e_11ee_b355_548d5a571682row11_col0,#T_ef86a5cb_596e_11ee_b355_548d5a571682row12_col0,#T_ef86a5cb_596e_11ee_b355_548d5a571682row12_col1{
            color:  black;
        }#T_ef86a5cb_596e_11ee_b355_548d5a571682row1_col1,#T_ef86a5cb_596e_11ee_b355_548d5a571682row3_col1,#T_ef86a5cb_596e_11ee_b355_548d5a571682row5_col1,#T_ef86a5cb_596e_11ee_b355_548d5a571682row7_col1,#T_ef86a5cb_596e_11ee_b355_548d5a571682row11_col1{
            color:  red;
        }</style><table id="T_ef86a5cb_596e_11ee_b355_548d5a571682" ><thead>    <tr>        <th class="blank level0" ></th>        <th class="col_heading level0 col0" >var2</th>        <th class="col_heading level0 col1" >onemorevar</th>    </tr></thead><tbody>
                <tr>
                        <th id="T_ef86a5cb_596e_11ee_b355_548d5a571682level0_row0" class="row_heading level0 row0" >0</th>
                        <td id="T_ef86a5cb_596e_11ee_b355_548d5a571682row0_col0" class="data row0 col0" >1</td>
                        <td id="T_ef86a5cb_596e_11ee_b355_548d5a571682row0_col1" class="data row0 col1" >0.457000</td>
            </tr>
            <tr>
                        <th id="T_ef86a5cb_596e_11ee_b355_548d5a571682level0_row1" class="row_heading level0 row1" >1</th>
                        <td id="T_ef86a5cb_596e_11ee_b355_548d5a571682row1_col0" class="data row1 col0" >2</td>
                        <td id="T_ef86a5cb_596e_11ee_b355_548d5a571682row1_col1" class="data row1 col1" >-3.815000</td>
            </tr>
            <tr>
                        <th id="T_ef86a5cb_596e_11ee_b355_548d5a571682level0_row2" class="row_heading level0 row2" >2</th>
                        <td id="T_ef86a5cb_596e_11ee_b355_548d5a571682row2_col0" class="data row2 col0" >2</td>
                        <td id="T_ef86a5cb_596e_11ee_b355_548d5a571682row2_col1" class="data row2 col1" >3.815000</td>
            </tr>
            <tr>
                        <th id="T_ef86a5cb_596e_11ee_b355_548d5a571682level0_row3" class="row_heading level0 row3" >3</th>
                        <td id="T_ef86a5cb_596e_11ee_b355_548d5a571682row3_col0" class="data row3 col0" >2</td>
                        <td id="T_ef86a5cb_596e_11ee_b355_548d5a571682row3_col1" class="data row3 col1" >-2.111000</td>
            </tr>
            <tr>
                        <th id="T_ef86a5cb_596e_11ee_b355_548d5a571682level0_row4" class="row_heading level0 row4" >4</th>
                        <td id="T_ef86a5cb_596e_11ee_b355_548d5a571682row4_col0" class="data row4 col0" >2</td>
                        <td id="T_ef86a5cb_596e_11ee_b355_548d5a571682row4_col1" class="data row4 col1" >2.111000</td>
            </tr>
            <tr>
                        <th id="T_ef86a5cb_596e_11ee_b355_548d5a571682level0_row5" class="row_heading level0 row5" >5</th>
                        <td id="T_ef86a5cb_596e_11ee_b355_548d5a571682row5_col0" class="data row5 col0" >2</td>
                        <td id="T_ef86a5cb_596e_11ee_b355_548d5a571682row5_col1" class="data row5 col1" >-1.222000</td>
            </tr>
            <tr>
                        <th id="T_ef86a5cb_596e_11ee_b355_548d5a571682level0_row6" class="row_heading level0 row6" >6</th>
                        <td id="T_ef86a5cb_596e_11ee_b355_548d5a571682row6_col0" class="data row6 col0" >2</td>
                        <td id="T_ef86a5cb_596e_11ee_b355_548d5a571682row6_col1" class="data row6 col1" >1.222000</td>
            </tr>
            <tr>
                        <th id="T_ef86a5cb_596e_11ee_b355_548d5a571682level0_row7" class="row_heading level0 row7" >7</th>
                        <td id="T_ef86a5cb_596e_11ee_b355_548d5a571682row7_col0" class="data row7 col0" >2</td>
                        <td id="T_ef86a5cb_596e_11ee_b355_548d5a571682row7_col1" class="data row7 col1" >-3.333000</td>
            </tr>
            <tr>
                        <th id="T_ef86a5cb_596e_11ee_b355_548d5a571682level0_row8" class="row_heading level0 row8" >8</th>
                        <td id="T_ef86a5cb_596e_11ee_b355_548d5a571682row8_col0" class="data row8 col0" >2</td>
                        <td id="T_ef86a5cb_596e_11ee_b355_548d5a571682row8_col1" class="data row8 col1" >3.333000</td>
            </tr>
            <tr>
                        <th id="T_ef86a5cb_596e_11ee_b355_548d5a571682level0_row9" class="row_heading level0 row9" >9</th>
                        <td id="T_ef86a5cb_596e_11ee_b355_548d5a571682row9_col0" class="data row9 col0" >2</td>
                        <td id="T_ef86a5cb_596e_11ee_b355_548d5a571682row9_col1" class="data row9 col1" >4.333000</td>
            </tr>
            <tr>
                        <th id="T_ef86a5cb_596e_11ee_b355_548d5a571682level0_row10" class="row_heading level0 row10" >10</th>
                        <td id="T_ef86a5cb_596e_11ee_b355_548d5a571682row10_col0" class="data row10 col0" >2</td>
                        <td id="T_ef86a5cb_596e_11ee_b355_548d5a571682row10_col1" class="data row10 col1" >4.444000</td>
            </tr>
            <tr>
                        <th id="T_ef86a5cb_596e_11ee_b355_548d5a571682level0_row11" class="row_heading level0 row11" >11</th>
                        <td id="T_ef86a5cb_596e_11ee_b355_548d5a571682row11_col0" class="data row11 col0" >2</td>
                        <td id="T_ef86a5cb_596e_11ee_b355_548d5a571682row11_col1" class="data row11 col1" >-5.555000</td>
            </tr>
            <tr>
                        <th id="T_ef86a5cb_596e_11ee_b355_548d5a571682level0_row12" class="row_heading level0 row12" >12</th>
                        <td id="T_ef86a5cb_596e_11ee_b355_548d5a571682row12_col0" class="data row12 col0" >2</td>
                        <td id="T_ef86a5cb_596e_11ee_b355_548d5a571682row12_col1" class="data row12 col1" >5.555000</td>
            </tr>
    </tbody></table>


def color_negative(v, color):
    return f"color: {color};" if v < 0 else None
df = pd.DataFrame(np.random.randn(5, 2), columns=["A", "B"])
df.style.map(color_negative, color='red')  
## Using matplotlib to add some color!


```python
poem = """
'Twas the night before Christmas, when all through the house
Not a creature was stirring, not even a mouse;
The stockings were hung by the chimney with care,
In hopes that St. Nicholas soon would be there;
"""
words = re.split(r'\W+', poem)
pairs = [(x, len(x)) for x in words]
# Create a 5x5 matrix of tuples: (word, length)
matrix = []
for i in range(0,5):
    matrix.append(pairs[5*i:5*(i+1)])
df2 = pd.DataFrame(matrix)

def font_short(v, weight, color):
    """
    Mark short words
    
    Parameters
    ----------
    v: tuple of (word, length)
    weight: CSS term accepted as a font-weight value
    color: CSS term accepted as a color
    """
    return f"font-weight: {weight}; color: {color}" if v[1] < 4 else None

# We must apply both `format` and `applymap` to the DataFrame.style
styler = df2.style.format(lambda x: x[0])
styler.applymap(font_short, weight='bold', color='magenta')
```




<style  type="text/css" >
#T_f9f93287_596e_11ee_af17_548d5a571682row0_col0,#T_f9f93287_596e_11ee_af17_548d5a571682row0_col2,#T_f9f93287_596e_11ee_af17_548d5a571682row1_col2,#T_f9f93287_596e_11ee_af17_548d5a571682row1_col4,#T_f9f93287_596e_11ee_af17_548d5a571682row2_col1,#T_f9f93287_596e_11ee_af17_548d5a571682row2_col2,#T_f9f93287_596e_11ee_af17_548d5a571682row2_col4,#T_f9f93287_596e_11ee_af17_548d5a571682row3_col1,#T_f9f93287_596e_11ee_af17_548d5a571682row3_col3,#T_f9f93287_596e_11ee_af17_548d5a571682row4_col0,#T_f9f93287_596e_11ee_af17_548d5a571682row4_col4{
            font-weight:  bold;
             color:  magenta;
        }</style><table id="T_f9f93287_596e_11ee_af17_548d5a571682" ><thead>    <tr>        <th class="blank level0" ></th>        <th class="col_heading level0 col0" >0</th>        <th class="col_heading level0 col1" >1</th>        <th class="col_heading level0 col2" >2</th>        <th class="col_heading level0 col3" >3</th>        <th class="col_heading level0 col4" >4</th>    </tr></thead><tbody>
                <tr>
                        <th id="T_f9f93287_596e_11ee_af17_548d5a571682level0_row0" class="row_heading level0 row0" >0</th>
                        <td id="T_f9f93287_596e_11ee_af17_548d5a571682row0_col0" class="data row0 col0" ></td>
                        <td id="T_f9f93287_596e_11ee_af17_548d5a571682row0_col1" class="data row0 col1" >Twas</td>
                        <td id="T_f9f93287_596e_11ee_af17_548d5a571682row0_col2" class="data row0 col2" >the</td>
                        <td id="T_f9f93287_596e_11ee_af17_548d5a571682row0_col3" class="data row0 col3" >night</td>
                        <td id="T_f9f93287_596e_11ee_af17_548d5a571682row0_col4" class="data row0 col4" >before</td>
            </tr>
            <tr>
                        <th id="T_f9f93287_596e_11ee_af17_548d5a571682level0_row1" class="row_heading level0 row1" >1</th>
                        <td id="T_f9f93287_596e_11ee_af17_548d5a571682row1_col0" class="data row1 col0" >Christmas</td>
                        <td id="T_f9f93287_596e_11ee_af17_548d5a571682row1_col1" class="data row1 col1" >when</td>
                        <td id="T_f9f93287_596e_11ee_af17_548d5a571682row1_col2" class="data row1 col2" >all</td>
                        <td id="T_f9f93287_596e_11ee_af17_548d5a571682row1_col3" class="data row1 col3" >through</td>
                        <td id="T_f9f93287_596e_11ee_af17_548d5a571682row1_col4" class="data row1 col4" >the</td>
            </tr>
            <tr>
                        <th id="T_f9f93287_596e_11ee_af17_548d5a571682level0_row2" class="row_heading level0 row2" >2</th>
                        <td id="T_f9f93287_596e_11ee_af17_548d5a571682row2_col0" class="data row2 col0" >house</td>
                        <td id="T_f9f93287_596e_11ee_af17_548d5a571682row2_col1" class="data row2 col1" >Not</td>
                        <td id="T_f9f93287_596e_11ee_af17_548d5a571682row2_col2" class="data row2 col2" >a</td>
                        <td id="T_f9f93287_596e_11ee_af17_548d5a571682row2_col3" class="data row2 col3" >creature</td>
                        <td id="T_f9f93287_596e_11ee_af17_548d5a571682row2_col4" class="data row2 col4" >was</td>
            </tr>
            <tr>
                        <th id="T_f9f93287_596e_11ee_af17_548d5a571682level0_row3" class="row_heading level0 row3" >3</th>
                        <td id="T_f9f93287_596e_11ee_af17_548d5a571682row3_col0" class="data row3 col0" >stirring</td>
                        <td id="T_f9f93287_596e_11ee_af17_548d5a571682row3_col1" class="data row3 col1" >not</td>
                        <td id="T_f9f93287_596e_11ee_af17_548d5a571682row3_col2" class="data row3 col2" >even</td>
                        <td id="T_f9f93287_596e_11ee_af17_548d5a571682row3_col3" class="data row3 col3" >a</td>
                        <td id="T_f9f93287_596e_11ee_af17_548d5a571682row3_col4" class="data row3 col4" >mouse</td>
            </tr>
            <tr>
                        <th id="T_f9f93287_596e_11ee_af17_548d5a571682level0_row4" class="row_heading level0 row4" >4</th>
                        <td id="T_f9f93287_596e_11ee_af17_548d5a571682row4_col0" class="data row4 col0" >The</td>
                        <td id="T_f9f93287_596e_11ee_af17_548d5a571682row4_col1" class="data row4 col1" >stockings</td>
                        <td id="T_f9f93287_596e_11ee_af17_548d5a571682row4_col2" class="data row4 col2" >were</td>
                        <td id="T_f9f93287_596e_11ee_af17_548d5a571682row4_col3" class="data row4 col3" >hung</td>
                        <td id="T_f9f93287_596e_11ee_af17_548d5a571682row4_col4" class="data row4 col4" >by</td>
            </tr>
    </tbody></table>




```python
# import matplotlib as mpl
def make_gradient(v, min_length, max_length, cmap='PuRd'):
    """
        Parameters
        ----------

        v: tuple of (word, length)
        min_length: int 
            minimum length of all words in the matrix
        max_length: int
            maximum length of all words in the matrix
        cmap: matplotlib color map, default value here is 'YlGn'

        Returns
        -------

        string:
            CSS setting a colour

        For Matplotlib colormaps:
        See: https://matplotlib.org/stable/tutorials/colors/colormaps.html
    """
    # normalize the word length as a fraction of the range 
    # between min_length and max_length
    rel_v = (v[1] - min_length) / (max_length - min_length)
    
    # define the colormap
    cmap = mpl.cm.get_cmap(cmap)
    
    # Get a colour out of the given colormap based on a value [0,1]
    rgba = cmap(rel_v)  
    
    # convert the colour to a hexadecimal string representation
    return f'background-color: {mpl.colors.rgb2hex(rgba)};'

# plot_color_gradients('Sequential',
#                      ['Greys', 'Purples', 'Blues', 'Greens', 'Oranges', 'Reds',
#                       'YlOrBr', 'YlOrRd', 'OrRd', 'PuRd', 'RdPu', 'BuPu',
#                       'GnBu', 'PuBu', 'YlGnBu', 'PuBuGn', 'BuGn', 'YlGn'])

# We must apply both `format` and `applymap` to the DataFrame.style
styler = df2.style.format(lambda x: x[0])
min_length = min([x[1] for x in pairs])
max_length = max([x[1] for x in pairs])
styler.applymap(lambda x: make_gradient(x, min_length, max_length, 'PuBuGn'))
```




<style  type="text/css" >
#T_a595eb72_596f_11ee_8729_548d5a571682row0_col0{
            background-color:  #fff7fb;
        }#T_a595eb72_596f_11ee_8729_548d5a571682row0_col1,#T_a595eb72_596f_11ee_8729_548d5a571682row1_col1,#T_a595eb72_596f_11ee_8729_548d5a571682row3_col2,#T_a595eb72_596f_11ee_8729_548d5a571682row4_col2,#T_a595eb72_596f_11ee_8729_548d5a571682row4_col3{
            background-color:  #84b2d4;
        }#T_a595eb72_596f_11ee_8729_548d5a571682row0_col2,#T_a595eb72_596f_11ee_8729_548d5a571682row1_col2,#T_a595eb72_596f_11ee_8729_548d5a571682row1_col4,#T_a595eb72_596f_11ee_8729_548d5a571682row2_col1,#T_a595eb72_596f_11ee_8729_548d5a571682row2_col4,#T_a595eb72_596f_11ee_8729_548d5a571682row3_col1,#T_a595eb72_596f_11ee_8729_548d5a571682row4_col0{
            background-color:  #b4c4df;
        }#T_a595eb72_596f_11ee_8729_548d5a571682row0_col3,#T_a595eb72_596f_11ee_8729_548d5a571682row2_col0,#T_a595eb72_596f_11ee_8729_548d5a571682row3_col4{
            background-color:  #519ec8;
        }#T_a595eb72_596f_11ee_8729_548d5a571682row0_col4{
            background-color:  #258bae;
        }#T_a595eb72_596f_11ee_8729_548d5a571682row1_col0,#T_a595eb72_596f_11ee_8729_548d5a571682row4_col1{
            background-color:  #014636;
        }#T_a595eb72_596f_11ee_8729_548d5a571682row1_col3{
            background-color:  #027c7e;
        }#T_a595eb72_596f_11ee_8729_548d5a571682row2_col2,#T_a595eb72_596f_11ee_8729_548d5a571682row3_col3{
            background-color:  #eee5f1;
        }#T_a595eb72_596f_11ee_8729_548d5a571682row2_col3,#T_a595eb72_596f_11ee_8729_548d5a571682row3_col0{
            background-color:  #016755;
        }#T_a595eb72_596f_11ee_8729_548d5a571682row4_col4{
            background-color:  #d7d5e8;
        }</style><table id="T_a595eb72_596f_11ee_8729_548d5a571682" ><thead>    <tr>        <th class="blank level0" ></th>        <th class="col_heading level0 col0" >0</th>        <th class="col_heading level0 col1" >1</th>        <th class="col_heading level0 col2" >2</th>        <th class="col_heading level0 col3" >3</th>        <th class="col_heading level0 col4" >4</th>    </tr></thead><tbody>
                <tr>
                        <th id="T_a595eb72_596f_11ee_8729_548d5a571682level0_row0" class="row_heading level0 row0" >0</th>
                        <td id="T_a595eb72_596f_11ee_8729_548d5a571682row0_col0" class="data row0 col0" ></td>
                        <td id="T_a595eb72_596f_11ee_8729_548d5a571682row0_col1" class="data row0 col1" >Twas</td>
                        <td id="T_a595eb72_596f_11ee_8729_548d5a571682row0_col2" class="data row0 col2" >the</td>
                        <td id="T_a595eb72_596f_11ee_8729_548d5a571682row0_col3" class="data row0 col3" >night</td>
                        <td id="T_a595eb72_596f_11ee_8729_548d5a571682row0_col4" class="data row0 col4" >before</td>
            </tr>
            <tr>
                        <th id="T_a595eb72_596f_11ee_8729_548d5a571682level0_row1" class="row_heading level0 row1" >1</th>
                        <td id="T_a595eb72_596f_11ee_8729_548d5a571682row1_col0" class="data row1 col0" >Christmas</td>
                        <td id="T_a595eb72_596f_11ee_8729_548d5a571682row1_col1" class="data row1 col1" >when</td>
                        <td id="T_a595eb72_596f_11ee_8729_548d5a571682row1_col2" class="data row1 col2" >all</td>
                        <td id="T_a595eb72_596f_11ee_8729_548d5a571682row1_col3" class="data row1 col3" >through</td>
                        <td id="T_a595eb72_596f_11ee_8729_548d5a571682row1_col4" class="data row1 col4" >the</td>
            </tr>
            <tr>
                        <th id="T_a595eb72_596f_11ee_8729_548d5a571682level0_row2" class="row_heading level0 row2" >2</th>
                        <td id="T_a595eb72_596f_11ee_8729_548d5a571682row2_col0" class="data row2 col0" >house</td>
                        <td id="T_a595eb72_596f_11ee_8729_548d5a571682row2_col1" class="data row2 col1" >Not</td>
                        <td id="T_a595eb72_596f_11ee_8729_548d5a571682row2_col2" class="data row2 col2" >a</td>
                        <td id="T_a595eb72_596f_11ee_8729_548d5a571682row2_col3" class="data row2 col3" >creature</td>
                        <td id="T_a595eb72_596f_11ee_8729_548d5a571682row2_col4" class="data row2 col4" >was</td>
            </tr>
            <tr>
                        <th id="T_a595eb72_596f_11ee_8729_548d5a571682level0_row3" class="row_heading level0 row3" >3</th>
                        <td id="T_a595eb72_596f_11ee_8729_548d5a571682row3_col0" class="data row3 col0" >stirring</td>
                        <td id="T_a595eb72_596f_11ee_8729_548d5a571682row3_col1" class="data row3 col1" >not</td>
                        <td id="T_a595eb72_596f_11ee_8729_548d5a571682row3_col2" class="data row3 col2" >even</td>
                        <td id="T_a595eb72_596f_11ee_8729_548d5a571682row3_col3" class="data row3 col3" >a</td>
                        <td id="T_a595eb72_596f_11ee_8729_548d5a571682row3_col4" class="data row3 col4" >mouse</td>
            </tr>
            <tr>
                        <th id="T_a595eb72_596f_11ee_8729_548d5a571682level0_row4" class="row_heading level0 row4" >4</th>
                        <td id="T_a595eb72_596f_11ee_8729_548d5a571682row4_col0" class="data row4 col0" >The</td>
                        <td id="T_a595eb72_596f_11ee_8729_548d5a571682row4_col1" class="data row4 col1" >stockings</td>
                        <td id="T_a595eb72_596f_11ee_8729_548d5a571682row4_col2" class="data row4 col2" >were</td>
                        <td id="T_a595eb72_596f_11ee_8729_548d5a571682row4_col3" class="data row4 col3" >hung</td>
                        <td id="T_a595eb72_596f_11ee_8729_548d5a571682row4_col4" class="data row4 col4" >by</td>
            </tr>
    </tbody></table>




```python
# Create one dataframe with only words
df_words = df2.applymap(lambda x: x[0])
# .. and a second one with only lengths
df_lengths = df2.applymap(lambda x: x[1])
# Now use the lengths to apply the background gradient to the words
df_words.style.background_gradient(gmap=df_lengths, axis=None)
```


    ---------------------------------------------------------------------------

    ValueError                                Traceback (most recent call last)

    C:\ProgramData\Anaconda3\lib\site-packages\IPython\core\formatters.py in __call__(self, obj)
        343             method = get_real_method(obj, self.print_method)
        344             if method is not None:
    --> 345                 return method()
        346             return None
        347         else:
    

    C:\ProgramData\Anaconda3\lib\site-packages\pandas\io\formats\style.py in _repr_html_(self)
        191         Hooks into Jupyter notebook rich display system.
        192         """
    --> 193         return self.render()
        194 
        195     @doc(NDFrame.to_excel, klass="Styler")
    

    C:\ProgramData\Anaconda3\lib\site-packages\pandas\io\formats\style.py in render(self, **kwargs)
        538         * table_attributes
        539         """
    --> 540         self._compute()
        541         # TODO: namespace all the pandas keys
        542         d = self._translate()
    

    C:\ProgramData\Anaconda3\lib\site-packages\pandas\io\formats\style.py in _compute(self)
        623         r = self
        624         for func, args, kwargs in self._todo:
    --> 625             r = func(self)(*args, **kwargs)
        626         return r
        627 
    

    C:\ProgramData\Anaconda3\lib\site-packages\pandas\io\formats\style.py in _apply(self, func, axis, subset, **kwargs)
        640             result.columns = data.columns
        641         else:
    --> 642             result = func(data, **kwargs)
        643             if not isinstance(result, pd.DataFrame):
        644                 raise TypeError(
    

    C:\ProgramData\Anaconda3\lib\site-packages\pandas\io\formats\style.py in _background_gradient(s, cmap, low, high, text_color_threshold, vmin, vmax)
       1124 
       1125         with _mpl(Styler.background_gradient) as (plt, colors):
    -> 1126             smin = np.nanmin(s.to_numpy()) if vmin is None else vmin
       1127             smax = np.nanmax(s.to_numpy()) if vmax is None else vmax
       1128             rng = smax - smin
    

    <__array_function__ internals> in nanmin(*args, **kwargs)
    

    C:\ProgramData\Anaconda3\lib\site-packages\numpy\lib\nanfunctions.py in nanmin(a, axis, out, keepdims)
        317         # Fast, but not safe for subclasses of ndarray, or object arrays,
        318         # which do not implement isnan (gh-9009), or fmin correctly (gh-8975)
    --> 319         res = np.fmin.reduce(a, axis=axis, out=out, **kwargs)
        320         if np.isnan(res).any():
        321             warnings.warn("All-NaN slice encountered", RuntimeWarning,
    

    ValueError: zero-size array to reduction operation fmin which has no identity





    <pandas.io.formats.style.Styler at 0x215f7c46fa0>




```python

```

