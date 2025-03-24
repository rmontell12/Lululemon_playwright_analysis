# Analysis of Lululemon Sale Section
Ruthie Montella

I have been a customer of Lululemon for many years now and have spent
more time scrolling on their website than I would like to admit. The
brand is considered to be a high-end yoga and exercise apparel company,
which is reflected in their high prices. To try and find more reasonable
prices I frequently visit the company’s sale page when I’m on their
website, or as they call it, the “We Made Too Much” section. This habit
peaked my interest as to what insights I could reveal with the data
available on Lululemon’s sale page, such as how much of a markdown is
actually applied to their products, since sometimes it doesn’t seem very
significant at all. To begin this analysis I used Playwright to scrape
the appropriate data from the women’s Lululemon sale page. I changed my
mind on what data I wanted to scrape a couple of times, but I ultimately
collected product name, sale price and regular price in my inital data
frame.

Code for this web scraping process - technically pulled 2 items off the
webpage (product name and then sale price and regular price which were
pulled as one item under the same html identiier):

``` python
import re
import pandas as pd
from playwright.sync_api import sync_playwright
import time
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns

# We Made Too Much Page - get item name, sale price and original price 

pw = sync_playwright()
pw = pw.start()
chrome = pw.chromium.launch(headless=False)
page = chrome.new_page()

page.goto('https://shop.lululemon.com/c/women-we-made-too-much/n16o10z8mhd')
#prod_select = page.locator('a[href*="prod"]') 

# Wait for the modal or promotion to appear and click the close button
# (Replace the selector with the actual CSS selector for the close button)
page.wait_for_selector('button.closeButton-1vSmX')  # Replace with the correct CSS selector for the close button
page.locator('button.closeButton-1vSmX').click()


def load_more_products(page, iterations=5):
    for _ in range(iterations): 
        product_names = []
        prices = []

        # Scroll to the bottom to make sure the "Load More" button is visible
        page.evaluate("window.scrollTo(0, document.body.scrollHeight)")
        time.sleep(np.random.uniform(0.2, 0.8)) 

        # Check if the "Load More" button is present and click it
        load_more_button = page.locator('button.button___rznsH.lll-text-button.pagination_button__V8a85.buttonTertiary__zgPUry')
        load_more_button.click()  # Click the "Load More" button

        product_name_locator = page.locator('a.link.lll-font-weight-medium')
        sale_price_locator = page.locator('.price') # need to access this
        #original_price_locator = page.locator('.price-compare') # and this 
        
        # Get the product details from the current page
        product_names += [product_name_locator.nth(i).inner_text() for i in range(product_name_locator.count())]
    
        prices += [sale_price_locator.nth(i).inner_text() for i in range(sale_price_locator.count())]
        #original_prices += [original_price_locator.nth(0).inner_text() for i in range(original_price_locator.count())]
    
    return product_names, prices


# Collect the product info by loading more products
product_names, prices = load_more_products(page, iterations=5)
    
print("Product Names:", product_names)
print("Prices:", prices)
product_names[0]
prices[0]

page.close()
chrome.close()
pw.stop()
```

After retrieving these values I began the lengthy process of cleaning
the data into the format I was looking for. First I had to split the
sale and regular prices into two different lists. I then tried to remove
as many extra characters as possible from around the numbers. Then I had
to handle the issue of many of the prices in both the sale and regular
categories having a range format, i.e. `$10-$20`. Obviously this format
is present to reflect the price range for the same item, but in
different colors which have been priced differently. Ultimately I
decided to take the average of the price range to use in my analysis.
Once this issue was handled I then looped through all the prices to make
sure all extra characters, such as `$` were removed and that the sale
and regular price columns were floats. Later on I also realized I needed
to clean the product name values to remove any newline characters from
the names so I included that function at the end as well. Hindsight I
recognize that I performed some of this data cleaning in a round about
way and certainly could have done it in fewer functions, but at the time
it was much easier to handle one issue at a time and just have one
function dedicated to handling each one.

Functions for data cleaning:

``` python
# now split the prices list into sale and regular 
def clean_prices(prices):
    sale_prices = []
    regular_prices = []
    
    for price in prices:
        # Split the string at the point where "Regular Price" begins
        try:
            sale_price, regular_price = price.split('Regular Price\xa0')
            sale_prices.append(sale_price.replace('Sale Price\xa0\n', '').strip())  # Clean and add sale price
            regular_prices.append(regular_price.strip())  # Clean and add regular price
        except ValueError:
            # If the 'Regular Price' is not found, add None to regular_prices
            sale_prices.append(price.replace('Sale Price\xa0\n', '').strip())
            regular_prices.append(None)

    return sale_prices, regular_prices


cleaned_sale_prices, cleaned_regular_prices = clean_prices(prices)


# some products have sale and regular prices formatted like so: $10-$50, making it hard to clean the
# data and to manipulate it. for the purposes of my project I am going to take the average sale or regular
# price for each item that has a range 

def clean_prices(prices): # removing all non $ or numerical characters
    cleaned_prices = []
    for price in prices:
        cleaned_price = price.replace('Sale Price\xa0\n', '').replace('Regular Price\xa0\n', '').strip()
        cleaned_prices.append(cleaned_price)
    return cleaned_prices


def clean_prices_with_range(prices): #handling price ranges - cleaning, getting both numbers and returning the average price
    cleaned_prices = []

    for price in prices:
        price_match = re.search(r'\$([\d,]+(?:\.\d{1,2})?)(?:\s*-\s*\$([\d,]+(?:\.\d{1,2})?))?', price)
        
        if price_match:
            min_price = price_match.group(1).replace(',', '')  # Remove commas from price
            max_price = price_match.group(2)
            
            if max_price:
                # Keep the range format and remove \xa0
                cleaned_prices.append(f'${min_price}-{max_price.replace(",", "")}')
            else:
                cleaned_prices.append(f'${min_price}')
        else:
            cleaned_prices.append(price.strip().replace("\xa0", ""))  # In case there's no range, clean price

    return cleaned_prices


# Separate sale and regular prices
sale_prices = []
regular_prices = []

for item in prices:
    sale_price_part = item.split('Regular Price\xa0\n')[0].replace('Sale Price\xa0\n', '').strip()
    regular_price_part = item.split('Regular Price\xa0\n')[1].strip() if 'Regular Price' in item else ''
    
    sale_prices.append(sale_price_part)
    regular_prices.append(regular_price_part)


# Clean the sale and regular prices using the clean_prices_with_range function
cleaned_sale_prices = clean_prices_with_range(sale_prices)
cleaned_regular_prices = clean_prices_with_range(regular_prices)

# Create data frame with cleaned sale prices and regular prices
df = pd.DataFrame({
    'Product Name': product_names,
    'Sale Price': cleaned_sale_prices,
    'Regular Price': cleaned_regular_prices
})

df['Product Name']


# function to remove $ on either side of of the dash character 
def clean_price_completely(price):
    if '-' in price:
        # If there's a range, split by '-' and calculate the average
        parts = price.replace('$', '').split('-')
        min_price = float(parts[0].strip())
        max_price = float(parts[1].strip())
        avg_price = np.mean([min_price, max_price])
        return float(round(avg_price, 2))
    else:
        # if no range, just remove the '$' and convert to float
        return float(price.replace('$', '').replace(',', '').strip())

# replace price columns with cleaned values and average values for ranges
df['Sale Price'] = df['Sale Price'].apply(clean_price_completely)
df['Regular Price'] = df['Regular Price'].apply(clean_price_completely)



# clean the product names - remove /n
def clean_product_name(product_name):
    # Remove line breaks and extra spaces
    product_name = product_name.replace('\n', ' ').strip()  # Replace newlines with spaces and strip any leading/trailing spaces
    return product_name

df['Product Name'] = df['Product Name'].apply(clean_product_name)
```

Once I created this data frame I was able to brainstorm a couple of
different questions that were of particular interest to me, the first
being the markdown percentage for each item. For this I simply used the
columns to do a calculation and add on a new “Markdown Percentage”
column. As a starting point I then decided to find some summary
statistics for this markdown column and found:

- Average markdown percentage: 37%
- Minimum markdown percentage: 12.84% - Softstreme Cinch-Waist Full-Zip
  Jacket, regularly priced at \$148 and on sale for \$129
- Maximum markdown percentage: 67.24% - Essential Tank Top, regularly
  priced at \$58 and on sale for \$19

Honestly I was pleasantly suprised by this markdown percentage as I
think it is somewhat reasonable considering Lululemon’s sale section is
usually pretty stocked with items. I would be interested to compare this
average markdown percentage to the the sale section of other high end
workout brands to better determine how quality this deal is.

Analysis Components:

``` python
# create a column with markdown percentage 
df['Markdown Percentage'] = round(((df['Regular Price'] - df['Sale Price']) / df['Regular Price']) * 100, 2) 

# summary stats for markdown percentages 
# average markdown percentage - 37% 
(sum(df['Markdown Percentage'])/len(df['Markdown Percentage']))
# max markdown percentage - is 67.24% markdown 
df[df['Markdown Percentage'] == max(df['Markdown Percentage'])]
# min markdown percentage - 12.84%
df['Regular Price'][df['Markdown Percentage'] == min(df['Markdown Percentage'])]
```

The next thing I was interested in was putting the product names into
categories by clothing type (i.e. sweatshirts, shirts, leggings, shorts,
etc). In order to do this I wrote the function below, utilizing `if`
statements to check for key words in each product name and categorize
accordingly. I established the category names based off of the
categories that Lululemon uses on their website, but I did not use every
single one of them. For example, for simplicity sake I decided to put
all shirts into one category, whether long or short sleeve, which Lulu’s
website would have broken down into two separate groups. I then
completed a couple of calculations based on this new markdown percentage
column, specifically looking at the average markdown perentage by
category and also finding the count of items in each category.

``` python
# assign the product names to categories - combining some categories as I see fit 
def categorize_product(product_name):
    product_name = product_name.lower()  # Convert to lowercase for easier comparison
    
    if 'jacket' in product_name:
        return 'Jackets'
    elif 'short' in product_name:
        return 'Shorts'
    elif 'shirt' in product_name:
        return 'Shirts'
    elif 'legging' in product_name or 'tight' in product_name:
        return 'Leggings'
    elif 'hoodie' in product_name or 'sweatshirt' in product_name or 'pullover' in product_name:
        return 'Hoodies & Sweatshirts'
    elif 'pant' in product_name or 'trouser' in product_name:
        return 'Pants'
    elif 'jogger' in product_name:
        return 'Joggers'
    elif 'bra' in product_name:
        return 'Sports Bras'
    elif 'dress' in product_name:
        return 'Dresses'
    elif 'skirt' in product_name:
        return 'Skirt'
    elif 'accessory' in product_name or 'bag' in product_name:
        return 'Accessories'
    elif 'shoe' in product_name:
        return 'Shoes'
    elif 'tank' in product_name:
        return 'Tank Tops'
    else:
        return 'Accessories'  # Default to 'Accessories' for all other items


df['Category'] = df['Product Name'].apply(categorize_product)



# Calculate average markdown by category
average_markdown_by_category = df.groupby('Category')['Markdown Percentage'].mean()


avg_prices_by_category = df.groupby('Category')[['Sale Price', 'Regular Price', 'Markdown Percentage']].mean().reset_index()

# Group by 'Category' and get the count of each category
category_counts = df['Category'].value_counts()
```

Finally I wanted to create a couple of visualizations both for my
understanding and to use for my presentation (can be viewed on my
presentation slides):

``` python
# show these category counts in a bar chart
    # accessories are by far the most represented category in the sale section - kind of a catch all for all smaller items
        # probably don't buy accessories unless they are on sale 
    # tank tops and shorts
        # could be because its winter still - seasonal rotation of what is one sale
        # would be interesting to see if this changes during the summer 
    # only one dress is on sale but lulu only sells a handful of dress styles
    # compared to other categories of clothing 

# bar chart for the count of items in each category
category_counts.plot(kind='bar', color='skyblue')
plt.title('Frequency of Each Category')
plt.xlabel('Category')
plt.ylabel('Count')
plt.xticks(rotation=45, ha='right')
plt.tight_layout()
plt.show()



# identify outliers and look at distribution of markdown % 

# historgram of markdown percentages by category
plt.figure(figsize=(10,6))
sns.boxplot(data=df, x='Category', y='Markdown Percentage', palette='Set3')
plt.title('Markdown Percentage Distribution by Category')
plt.xlabel('Category')
plt.ylabel('Markdown Percentage (%)')
plt.xticks(rotation=45)
plt.savefig('~/Desktop/uda_notes/markdown_percentage_distribution.png', bbox_inches='tight')
plt.show()
```

Conclusions:

Based on these plots and my analysis overall I was able to determine
that roughly 25% of items within Lululemon’s sale section are items from
the accessories category, which makes sense considering these items
usually don’t sell as well as clothing. I also gained the insight that,
at least right now in February, tank tops and shorts are the next most
common items to find on sale. Not only are tank tops one of the most
frequently placed on sale, but the distribution of their markdown
percentage is also the largest – meaning if you get lucky you could find
a really good deal. Of course now knowing this information I am
definitely going to think twice about purchasing either of these items
for full price, especially during the colder months, since I know they
should have a decent selection on sale. I also noticed that only one
singular dress and two pairs of shoes were in the sale section. This
makes sense when we generally consider how many items of each category
Lululemon sells. They have many styles of leggings, shirts and shorts,
but they have far fewer options within their shoe and dress categories.

Interestingly, leggings have the highest average markdown percentage of
all the categories, suprising when considering that Lululemon’s leggings
are potenitally their most popular item. This brings me to an important
consideration which is how the color of an item impacts its markdown
percentage. Lululemon tends to release many of their items in a very
wide range of colors and patterns, some of which end up selling better
than others. It would be very interesting to examine how many sale items
are a solid color verses a pattern, which I could examine by scraping
the different color names and sorting them into two groups, based on my
own categorization of them, into printed or solid. Obviously the item
design/product name is not the sole determining factor in setting a sale
price, the popularity of the color of the item is another important
factor. While I was not able to factor this consideration into this
analysis, this would be a very interesting potential next step to
include.

Overall I really enjoyed this project and was excited to find some
practical use from it that I will definitely be applying to my shopping
habits in the future.
