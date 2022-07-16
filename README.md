# Crime in Chicago
Power BI report by Robert Nowak

This data presents number of crime in each police district in Chicago.<br>
[Power Bi report page 1](images/Chicago_crime_data_1.jpg)<br>
On the first page user can choose district from map of chicago to see number of crimes in given date range and also crime distribusion for top 10 crimes in chosen district.<br>
[Power Bi report page 2](images/Chicago_crime_data_2.jpg)<br>
On the second page is visualization of number of crimes distribusion of chosen crime type for chosen districts though years. Table shows changes in number of crimes for given year to previous one for chosen features.<br>
[Power Bi report page 3](images/Chicago_crime_data_3.jpg)<br>
On the third page is graph which shows distribusion of chosen type of crime though month for given years and bar charts with 5 top crimes for every combination of boolean parameters: arrested and domestic.<br>

Source of the data: https://data.cityofchicago.org/Public-Safety/Crimes-2001-to-Present/ijzp-q8t2<br>
Range of data: from 01.01.2016 to 31.12.2021

# Preparation of the data
Data was prepared in Jupyter Notebook - Crimes_in_chicago_data_preparation.ipynb. Each step and decision is described in this file.<br>
Prepared data was loaded into PostgreSQL database -  Crimes in Chicago - into tables: cases, location and crime. Relation between tables is shown on relations.png image.<br>
Cases containes unique case number, date, updated on date, arrest boolean, domestic boolean, location description and foregin keys.<br>
Location contains uniqe location_id indentyfier, district number, ward number, beat number and block adress.<br>
Crime contains uniqe IUCR code, primary type description and specific description.<br>
Inside PostgreSQL database 5 views were created for better understanding of the data.<br>
Tables were imported into the Power BI report.

# List of requirements
Power BI report:
- Power BI Desktop (verion used for this project 2.107.683.0 64-bit) or cloud app.powerbi.com<br>
Jupyter Notebook file:
- Jupyter Notebook 6.4.8
- pandas 1.4.2
- extract zip files inside the data folder<br>
PostgreSQL:
- postgresql 14