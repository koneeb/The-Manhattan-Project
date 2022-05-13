# the-manhattan-project
Team Good-2-Great Final project for Python, Data, and Databases at UT Austin Spring 2022.

Team: 
* [Pedro Rodrigues](https://github.com/PedroNBRodrigues) - Data Engineering
* [Kashaf Oneeb](https://github.com/koneeb) - Data Engineering
* [Austin Longoria](https://github.com/galongoria) - Data Engineering, Analysis
* [Arpan Chatterji](https://github.com/achatterji1) - Analysis
* [Colin McNally](https://github.com/cmcnally23) - Streamlit Front End Dev
* [Elliott Metzler](https://github.com/ElliottMetzler) - Project Management, Git Management, Report, Streamlit Back End Dev

Due: 5/13/2022

## Introduction

Ever find yourself hankering for a drink but too tired to get to the store? We get it, it's been a rough week, it's Friday night, and all you want to do is quickly whip something up based on what you already have in your cabinet. That's where we come in. Our goal with The Manhattan Project is to use data on over 500 cocktails and allow you to search based on what you have handy. The Manhattan Project tool will beam you up with a handy list of delicious drinks containing what you've got on hand and how much effort you've got left in you (how many ingredients to include). Cheers!

The remainder of this report is structured as follows: first, we discuss the data used for our analysis and applet; next, we discuss our analytical approach and results; finally, we conclude with closing thoughts on limitations of our analysis areas for potential extensions to this project and search engine.

## Database

[The Cocktail Database](https://www.thecocktaildb.com/) is "an open, crowd-sourced database of drinks and cocktails from around the world." The online database contains a total of 635 unique drinks ranging from classics like a [Manhattan](https://www.thecocktaildb.com/drink/11008-Manhattan) or a [Martini](https://www.thecocktaildb.com/drink/11728-Martini) to more peculiar concoctions like the [Brain Fart](https://www.thecocktaildb.com/drink/17120-Brain-Fart). It also contains listings for mixed shots like the [Lemon Shot](https://www.thecocktaildb.com/drink/12752-Lemon-Shot) or [Shot-gun](https://www.thecocktaildb.com/drink/16985-Shot-gun). 

Since the Cocktail Database (hereafter "cocktailDB") has a JSON API and richer extraction capabilities with a $2 fee to [Patreon](https://www.patreon.com/thedatadb) (not to be confused with [Patron](https://www.patrontequila.com/age-gate/age-gate.html?origin=%2F&flc=homepage&fln=Post_Homepage_Patron)), we purchased full access. We extracted and cleaned the data using Python, wrote schema in Postgres, and used Google Cloud Platform Buckets to upload csv files and import them into our own database.

The second important piece of data we used are ingredient prices from the web. We compiled a list of ingredients from the cocktailDB and leveraged [Walmart's API](https://www.bluecartapi.com) for drink prices. As with the data from the cocktailDB, we uploaded a table of ingredient prices to our database.

### Cocktail Data

The first step in retrieving our data was to query the cocktailDB API endpoints for each drink in their database. To do this, we looped over each letter of the alphabet and digits 0-9, querying the database for cocktails starting with each letter or number. This approach yielded all 635 cocktails and their complete information.

#### Description of the Cocktail Data

The raw data contains an entry (row) for each drink and many descriptive features (columns) about that drink. In addition to a unique ID for each drink and the drink name, it contains a handful of classification fields such as the the drink glass type (e.g. highball, shot glass, punch bowl, etc.) and whether or not the drink is alcoholic. Additionally, it contains fields containing written instructions in multiple languages including English, German, French and Italian. Most importantly for our purposes, each cocktail has a batch of columns reporting the ingredients required to make that cocktail and a corresponding set of columns reporting the measurements of those ingredients. In the raw data, these ingredient measurements are non-standard in format and unit, including entries such as "2 shots", "2L", or "3 parts." Since these measurements are non-standard, this represented the biggest data cleaning effort to prepare our data for analysis.

#### Cocktail Data Cleaning Process

Though we broadly attempted to upload the data to our database in as raw of a format as possible, we took some important cleaning steps. We dropped a few columns that we were certain not to use. Most importantly, we cleaned the ingredient measurements and converted these values to ounces so that we could estimate drink prices.

* The first step in our cleaning process was dropping columns with all missing values. These columns included the alternate drink name `strDrinkAlternate`, instructions in Spanish `strInstructionsES`, French `strInstructionsFR`, Chinese in simplified script `strInstructionsZH-HANS`, and Chinese in traditional script `strInstructionsZH-HANT`. Similarly, we noticed that none of the drinks had more than 12 ingredients, therefore, we dropped the ingredients and their respective measurement columns for ingredients 13 through 15 i.e. `strIngredient13`, `strIngredient14`, `strIngredient15`, `strMeasure13`, `strMeasure14`, and `strMeasure15`. 
* The next step was to remove rows that had none of the ingredient measurements specified, given that at least one ingredient was specified. We observed 7 such cases. For the remaining rows, we added the string "1 oz" for entries where the ingredient was specified but the respective measurement was not. We observed a total of 90 such cases. This step was necessary since we later convert missing values of measurements to "0" for the ingredients that were not specified.
* We then proceeded to create a total number of ingredients column `total_ingredients` that counted across the ingredient columns, 1 through 12 and returned the number of ingredients required to make the drink in the respective row. This column would be useful in specifying the number of ingredients needed to make a drink for our proposed applet.
* Next, we created 12 new "clean" ingredient measurement columns, `strMeasure1_clean` through `strMeasure12_clean` that hold cleaned strings from the actual ingredient measurement columns i.e. `strMeasure1` through `strMeasure12` with commas, and parentheses removed. We also removed specific words appearing before digits which included "Add", "Around rim put", "About", and "Juice of". This would allow us to convert strings to floats later in the cleaning process. 
* We then replaced new-line characters in the measurements `strMeasure1` through `strMeasure12`, instructions `strInstructions`, `strInstructionsDE`, and `strInstructionsIT`, and image attribution `strImageAttribution` columns with the space character to improve compilation and readability of the csv file. 
* Finally, we created a unit conversion dictionary with all the units specified in the "clean" measurement columns as keys and their respective measurement in ounces as values. We also converted fractions in the clean measurement columns to floats. Using regex and our unit conversion dictionary, we returned a csv with all the observations in the clean measurement columns converted to floats representing the ingredient measurements in ounces. This would prepare the measurement columns for the quantitative analysis. Note: our cleaning code returns two files: a csv with no headers to allow the data to fit the SQL table schema, and a csv with only headers as a reference to set up the SQL table schema.

### Ingredient Prices Data

To retrieve data on ingredient prices, we combined a list of ingredients from the cocktail database to search Walmart's API (dubbed BlueCart), performed a combination of programatic and manual cleaning on the ingredient prices, then imported the resulting data into our database. One key nuance to our approach to searching the BlueCart API was using two searches to achieve most representative results. We queried both for "best seller" and "best match" for each ingredient, then combined these two searches in an effort to get the closest, most accurate, representation of the price of that ingredient.

#### Description of the Ingredient Prices Data

The raw ingredient prices data returned from the BlueCart API included information on the ingredient name (based on our search parameters) and the price of the ingredient. However, this ingredient price was also not in standardized unit format, so as with the Cocktail Data we had to perform a cleaning process to convert units into a consistent format for use. Thus, in addition to the name of the product and the price, we retrieved the description to obtain the size of the portion so we could perform analysis to convert units.

#### Ingredient Prices Data Cleaning Process

To clean the Ingredient Prices Data, we utilized the descriptions for the products. We found that the measurement and units were consistently appearing at the end of the product description, so we iterated through descriptions in reverse order to extract the unit and portion size. For certain ingredients, we were unable to systematically perform conversions, so we implemented a manual verification and review step separately to fill in necessary missing entries and convert in certain cases. For our final step, we converted prices to a per ounce basis.

## Streamlit Applet

The first goal of this project was to use our database to produce a handy Streamlit applet to allow users to query for drinks using various search parameters. We allow users to query based on the following parameters:
* Ingredient Count: the maximum number of ingredients they have available or would be willing to include in their cocktail.
* Main Liquors: Some main liquors like Vodka, Whiskey, or Tequila to allow users to easily find recipes with common ingredients.
* Other Ingredients: Using a unique list of ingredients appearing in the database, we allow users to get specific if they have something they'd like to focus on or use up.

## Analysis [[Elliott left this alone for now]]

The second goal of this project was to use our database to perform some analysis of the cocktail data.

We began our analysis by looking at ingredient prevalence. To do so, we checked the top 10 most used ingredients among the entire dataset. Our results are shown below:

Table***


This table shows that only three types of liquor: rum, gin, and vodka are in at least 15% of the drinks. This was our first clue that some of the drinks in our dataset may not include liquor at all. Since we were interested in alcoholic cocktails... [[Note for Elliot - This is not our first clue that some drinks don't contain alcohol, we mentioned earlier in the database section that there are options for alcoholic or not, so it must be that some didn't have]]

Our next step was to determine the types of drinks we were interested in. The focus of our analysis became alcoholic beverages that include liquor. The 6 types of distilled spirits are: brandy, gin, rum, tequila, vodka, and whiskey, so we reduced our dataset to include drinks that include one or more of these types of alcohol. Next, we viewed the most used spirits in our list of cocktail recipes. The results can be seen below: [[This should be the first step]]

Table2***

somethin semthing ....i forgot what this graph looks like

At this point we began our price analysis. It was clear to us that the most expensive ingredients were the different types of liquor, so our next step was to compare the price per ounce of the different types of spirits, which can be seen in the chart below:

BarChart****


Comparing this chart with the previous table (Table2), we can see that the most expensive spirits are used the least often. Interestingly, the least expensive liquor (grain alcohol) is also used the least often. We acknowledge the fact that there may be sampling bias associated with these proportions. However, it should be noted that rum and vodka were subdivided into "rum" and "vodka" and "flavored rum" and "flavored vodka". Even so, the "rum" and "vodka" categories lie in the top 3 most used spirits. We assume this is either because rum, gin, and vodka mix well with other ingredients or people just like the taste of them. 


####

*****this will be before ols to say that there is no perfect collinearity**** part of our analysis included whether or not certain types of alcohol are mixed together. Our dataset at this point included drinks that were tens of ounces or more, so further subdivied our data into cocktails that are less than or equal to 8 ounces. This is a reasonable number for a person who is drinking a given cocktail in one sitting. We found that the correlation between spirits is negative but close to zero. Thus, typically single-person cocktails in our dataset are using only one type of liquor. The results are shown in a heat map below:

**heat map**


####

## Conclusion

[[Elliott to fill TODAY]]

* Limitations
* Summarize Findings
* Extensions (to do but also what data extensions would help)
[[
* We could have used a database that only included alcoholic cocktails and more drinks, as [Cocktails API](https://cocktailsapi.xyz).
* Create a new column with measurement and name of ingredient, then a dictionary based on measurement of the ingredients, instead of arbitrary value. For example, currently we convert "juice of orange" and "juice of lemon" to 1 oz, because they are defined as "juice of".
* The sizes of the drinks vary from punch bowls to shots and we could have scaled them to a single size.
* By being an open source, anyone with a key could edit or add new cocktails, which leads to contamination of the data, with random creations and with data entry errors.
* Expand parameters of streamlit, for example, add an option to specify quantity of each ingredient that you have available and are willing to use in a drink.
]]

## Reproducability Instructions

__NOTE__: [[Need to think about and verify what the system requirements are. We are going to have requirements for access to cloud buckets to upload the CSVs I think, along with GCP for normal stuff. Perhaps other for an app if we try to run one]]

1) Set-up Instructions:
    * Clone this repository to your local machine following the standard procedure of copying the SSH clone path and running `git clone <CLONE_PATH>`.
    * Run `cd the-manhattan-project`
    * Run `pip install -r requirements.txt` or `python3 -m pip install -r requirements.txt`, depending on your system.

2) Instructions to source and clean the data:
    * Run `python3 code/pull_raw_data.py` to scrape data from [The Cocktail DB](https://www.thecocktaildb.com) website and create two csv files: [drinks_data_raw.csv](https://github.com/ElliottMetzler/the-manhattan-project/blob/main/data/drinks_data_raw.csv) which contains raw data on drink recipes and [ingredients_data_raw.csv](https://github.com/ElliottMetzler/the-manhattan-project/blob/main/data/ingredients_data_raw.csv) which contains a list of all the ingredients specified in the recipes. The ingredient list will be utilized in pulling prices for the ingredients. Note that you will need to acquire an API key and insert it into the file. Specifically, replace the string "INSERT_API_KEY" with your API key. To acquire an API key, you can go to [The Cocktail DB Patreon](https://www.patreon.com/thedatadb). The $2 level supporter is more than enough for a key that gives access to all the drinks in the database and the ingredient list API search. After acquiring it, it should take up to an hour to receive it, keep in mind that the creators work under the GMT working hours, so it may take longer.
    * Run `python3 code/clean.py` to create two csv files: [drinks_data_clean_no_header.csv](https://github.com/ElliottMetzler/the-manhattan-project/blob/main/data/drinks_data_clean_no_header.csv) which contains clean data with no headers to faciliate merging it into the SQL table and [drinks_data_headers.csv](https://github.com/ElliottMetzler/the-manhattan-project/blob/main/data/drinks_data_headers.csv) to serve as a reference for setting up the SQL table schema.
    * Run `python3 code/prices_pull.py` to scrape data from the BlueCart API. Note that you will need to acquire an API key and insert it into the file in the parameters section. Specifically, replace the string "INSERT_API_KEY" with your API key. [[Note for Austin - write how to get the API key]]
    * Run `python3 code/prices_clean.py` [[Specify the files created by this code and add links]]
    * Run `python3 code/input_missing_prices.py`

3) Instructions to create the database:
    * Make a database instance in [Google Cloud Platform ("GCP")](https://cloud.google.com/). Go to GCP SQL and create a PostgreSQL 13 database instance, you can use the ["Create an instance"](https://console.cloud.google.com/sql/choose-instance-engine?project=deft-diode-342909) to do so. Make sure you whitelist the IPs in block `0.0.0.0/0` and select a password for it.
    * Create a database in GCP SQL and name it `drinks`. You can do that by going to the "Databases" tab in the newly created instance.
    * Connect to your database with DBeaver. Your host is the `Public IP Address` found in GCP SQL on the "Overview" tab. The port will be the default Postgres port: `5432` and the username is the default Postgres username: `postgres`, you don't have to change it. The password is the same password you created for the instance. The database you need to select is `drinks`.
    * In DBeaver, navigate to `drinks` > `databases` > `drinks`. Right-click the database `drinks`, then select `SQL Editor` > `New SQL Script`. 
    * Copy the commands from [create_tables.sql](https://github.com/ElliottMetzler/the-manhattan-project/blob/main/setup/create_tables.sql) into the SQL Script and execute it to create the database tables.
    * Create a bucket in GCP Cloud Storage. You can do that by accessing ["Cloud Storage"](https://console.cloud.google.com/storage/browser?_ga=2.133749006.1075698642.1652116044-1317346431.1646212364&_gac=1.195626590.1651155734.CjwKCAjw9qiTBhBbEiwAp-GE0Yk6cV8xAcydrJuB-bCw6AUvFJOwOvxnNvhWUdilN62kp9mxZnKz_hoCepoQAvD_BwE&project=deft-diode-342909&prefix=) on the GCP platform.
    * Upload the `drinks_data_clean_no_header.csv` and the `ingredient_prices_clean.csv` to the newly created bucket. 
    * Import the `drinks_data_clean_no_header.csv` from the bucket into the created table. To do so, you can go to GCP's SQL and use the "import" option, when prompted to choose a source, choose the CSV file from the bucket, with file format "CSV". For the "Destination", select the `drinks` database and the `all_cocktails` table. 
    * Next, import the `ingredient_prices_clean.csv` from the bucket into the created table. You should repeat the following step, but select the `ingredient_prices` table instead of the `all_cocktails` table.
    * Before you can run query commands, you must give it the right credentials to connect to your database. Copy the file demo.env to .env and modify it by providing the credentials you created above. An easy way to do this is to run `cp demo.env .env` and then modify the .env file.

4) Instructions to run the Streamlit app:
    * Run `streamlit run code/streamlitty.py`. [[Please specify what will happen after running the code]]
    * Open a browser and copy the url from the terminal to your browser search bar. [[Specify which URL, network or external]]
    * View and use the applet!
