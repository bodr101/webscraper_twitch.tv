<b>Import packages & run Selenium</b>

<i>Selenium</i> is used to run the webdriver and perform actions, such as scrolling, on the webdriver. <i>Time</i> is imported to set a sleeptime for the program to run, which stops the program for a predetermined period. <i>Datetime</i> is imported to give a specific timestamp to capture the time of collection for the scraper. <i>BeautifulSoup</i> is imported to find parse the HTML soup. <i>Re</i> is imported to adjust full streamer names to only the tags which can be used to make the URL of the twitch stream. <i>JSON </i> allows the scraper to store all of the data into a json file and <i>TQDM</i> helps visualizing the progress of the scraper.


```python
from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from webdriver_manager.chrome import ChromeDriverManager
from selenium.webdriver.common.keys import Keys
import time
from selenium.webdriver.common.by import By
from datetime import datetime
from selenium.webdriver import ActionChains
from bs4 import BeautifulSoup # Use beautifulsoup to parse the html soup
import re
import json
from tqdm.notebook import tqdm
```

# Chapter 1: Open Chromedriver and collecting categories 

<br>
This part of the scraper contains code to start the chromedriver and scrape the top 30 categories on that moment that are livestreamed on https://www.twitch.tv/directory.

## 1.1 Run Chromedriver

Here the chromedriver options are defined. The chromedriver is opened with default content settings to decrease loading time of the websites and the language of the chromedriver is set to english, to prevent category names to differ in different languages.


```python
chrome_options = webdriver.ChromeOptions()
chrome_options.add_experimental_option(
    #this will disable image loading
    "prefs", {"profile.managed_default_content_settings.images": 2,'intl.accept_languages': 'en,en_US'}
)
driver = webdriver.Chrome(ChromeDriverManager().install(), chrome_options=chrome_options)

url = "https://www.twitch.tv/directory?sort=VIEWER_COUNT"       
```

    C:\Users\Pepijn de Vries\AppData\Local\Temp\ipykernel_30128\867641456.py:6: DeprecationWarning: executable_path has been deprecated, please pass in a Service object
      driver = webdriver.Chrome(ChromeDriverManager().install(), chrome_options=chrome_options)
    C:\Users\Pepijn de Vries\AppData\Local\Temp\ipykernel_30128\867641456.py:6: DeprecationWarning: use options instead of chrome_options
      driver = webdriver.Chrome(ChromeDriverManager().install(), chrome_options=chrome_options)
    

## 1.2 Collecting Category names
<br>
In this code snippet the driver is opened on the browse page where all of the categories are listed. After opening up the driver page, the names of the categories are collected and put into a list.



```python
# Gets the URL in maximized window
driver.maximize_window();
driver.get(url)
time.sleep(5)

soup=BeautifulSoup(driver.page_source)
categories = soup.find_all('h2', class_ = "CoreText-sc-cpl358-0 fzONq")
category_list = []
for category in categories:
    category = category.get_text()
    category = category.replace(' ','%20')
    category = category.replace(':', '%3A')
    category_list.append(category)    
```

# Chapter 2: Generating a list of streamer names
<br>
The following function takes a category from <b>category_list</b> and opens the category website. The driver will then start scrolling untill the minimum number of viewers(which is predetermined) is reached. When the driver arrives at the minimum number of viewers, it will stop scrolling. <i>BeautifulSoup</i> will transform the HTML data to readable text and all of the streamer names will be collected and put into a list <b>streamer_name</b>. When the list <b>streamer_name</b> is compiled, the <i>get_twitch_data</i> function in the following chapter will go through all of the streamer names their 'about' pages. 


```python
def get_streamer_data(category_list):
    url = "https://www.twitch.tv/directory/game/" + category_list +"?sort=VIEWER_COUNT"
    driver.get(url)
    streamer_name = []
    time.sleep(2)
    # Here we create a variable check which indicates the while loop wheter to stop or not.
    check = True
    #Here you can fill in the minimum amount of viewers the streamer must have for the scroller to stop.
    minimum = 50
    # With this part we put the focus on an element on the page.
    element = driver.find_element(By.XPATH, '//*[@id="directory-game-main-content"]/div[1]/div[2]/div/div[2]/div[1]/h1')
    action = ActionChains(driver)
    action.move_to_element(to_element = element).click().perform()
    #Here we create a scrolling function which stops at the minimum amount of viewers, mentioned above.
    soup = BeautifulSoup(driver.page_source)
    viewer_list = soup.find_all(class_ = "ScMediaCardStatWrapper-sc-1ncw7wk-0 jluyAA tw-media-card-stat")
    print("Scrolling... (could take some time)\n")
    while (check == True):
        element2=driver.find_element(By.TAG_NAME,  'body')
        element2.send_keys(Keys.END)
        time.sleep(1)
        soup = BeautifulSoup(driver.page_source)
        viewer_list = soup.find_all(class_ = "ScMediaCardStatWrapper-sc-1ncw7wk-0 jluyAA tw-media-card-stat")
        for item in viewer_list:
            viewer_count = re.sub(r'[^0-9,.,K]', '', item.get_text())
            if "K" in viewer_count:
                viewer_count = re.sub(r'[^0-9,.,]', '',viewer_count)
                double_check = int(float(viewer_count) * 1000)
            else: 
                double_check = int(viewer_count)       
            if (double_check< minimum):
                    check = False
    
    print("Done Scrolling for streamers \n")
    
    time.sleep(2)
    #Reshape the categories to be printed as normal text.
    category_list = category_list.replace('%20',' ')
    category_list = category_list.replace('%3A', ':')
    time.sleep(2)
    #Collecting all of the streamer names that are loaded on the page and putting them into a list.
    soup=BeautifulSoup(driver.page_source)
    streamers = soup.find_all('p', class_ = "CoreText-sc-cpl358-0 eyuUlK")
    for streamer in streamers:    
        name = re.sub(r'[^a-zA-Z,_,0-9,(]', '', streamer.get_text())
        name = name.split("(")
        try:
            name = name[1]
        except:
            name = name[0]    
        streamer_name.append(name)
    #When the full list of streamer names is compiled, the function calls a different function to retreive all of the individual
    #streamer information
    for name in tqdm(streamer_name, desc ="Getting streamers from the Category: " + category_list, mininterval = 5):
        get_twitch_data(name)
    streamer_name.clear()
```

# Chapter 3: Collecting individual streamer information

<b>In this part of the code, a function has been written which collects the datapoints for all of the individual streamers. The datapoints collected per streamer are:</b>
1. Time of collection 
2. Name of the streamer
3. Name of the category (if applicable)
4. Name of the team of the streamer (if applicable)
5. Amount of followers of the streamer
6. Content block list, which includes every 'img' element from the 'about' page of the streamer
7. Textual content block, which includes every textual element from the 'about' page of the streamer

These datapoints are all appended to an empty list which is created at the start of the function, called: 'data'.

The final part of the code, 'json.dumps', dumps the dictionary filled with datapoints into the json file at the last part of the code, where a twitch_data.json file is openend.



```python
#This is the function that gets the information of the streamer's about page
def get_twitch_data(streamer_name):
    data = []
    url = "https://twitch.tv/" + streamer_name + "/about"
    driver.get(url)
    #We let the code sleep for two seconds, so it has time to load in the full page.
    time.sleep(2)
    soup=BeautifulSoup(driver.page_source)
    now = datetime.now()
    #This code collects the timestamp of collecting the data
    dt_string = now.strftime("%d/%m/%Y %H:%M:%S")
    # With this code we collect the name of the streamer
    try: 
        name_streamer = soup.find(class_ = "CoreText-sc-cpl358-0 ScTitleText-sc-1gsen4-0 cyfUN gasGNr tw-title").get_text()
    except:
        name_streamer = streamer_name
    # Printing the name of the category of the streamer
    try:
        category = soup.find(class_ = "CoreText-sc-cpl358-0 ScTitleText-sc-1gsen4-0 iUOVlD gasGNr tw-title").find_all('span')[1]
        category = category.get_text()
    #If a streamer is banned or stopped streaming between scraping their name from the category page 
    #and scraping their about page they will receive NA for category
    except:
        category = "NA"
    # With this code we print the teamname if there is one, otherwise it will fill in "NA" for team name
    try:
        team_name = soup.find(class_ = "CoreText-sc-cpl358-0 fIcpuT").get_text()
    except:
        team_name = "NA"
    
    # With this code we collect the amount of followers the streamer has
    try:
        follower_count = soup.find(class_ = "CoreText-sc-cpl358-0 gVjya").get_text()
    except:
        "Not Found"
    content_block = []
    for link in soup.find_all('img', {'data-test-selector':"image_test_selector"}): 
        content_block.append(link['src'])
    # With this code we collect the textual elements from the info page of a streamer    
    textual_content_block = []
    for el in soup.find_all(class_ = "Layout-sc-nxg1ff-0 ScTypeset-sc-xkayed-0 iwcdTx fbdVqS tw-typeset"):
        textual_content_block.append(el.find_all(text=True))
    # In this for loop we collect all the data from above and put it in a dictionary. Afterwards we dump this dictionary in a json-file
    data.append({'Time of collection': dt_string,
                'Name of the streamer': name_streamer,
                'Category': category,
                 'Team Name': team_name,
                'Followers': follower_count,
                'Content Block': content_block,
                'Textual Content Block': textual_content_block})
    for item in data:
        f.write(json.dumps(item))
        f.write('\n')
```

# Chapter 4: Write collected information to a .json file

In the final part of the code, a .json file is opened called 'twitch_data.json'. The information is stored in this file through a for loop which iterates over the entire list of categories. Within this categories function, the get_streamer_data function is located, which scrapes data of each individual streamer. This is then written to the .json file and holds the variables as a dictionary. This dictionary full of datapoints is dumped into the 'twitch_data.json' file through the 'json.dumps' code in the 'get_streamer_data' function.



```python
f = open('twitch_data.json', 'w',encoding = 'utf-8')    
for category in category_list:
    get_streamer_data(category)
f.close()
driver.close()
```

    Scrolling... (could take some time)
    
    Done Scrolling for streamers 
    
    


    Getting streamers from the Category: League of Legends:   0%|          | 0/182 [00:00<?, ?it/s]


    Scrolling... (could take some time)
    
    Done Scrolling for streamers 
    
    


    Getting streamers from the Category: Just Chatting:   0%|          | 0/673 [00:00<?, ?it/s]


    Scrolling... (could take some time)
    
    Done Scrolling for streamers 
    
    


    Getting streamers from the Category: Overwatch 2:   0%|          | 0/236 [00:00<?, ?it/s]


    Scrolling... (could take some time)
    
    Done Scrolling for streamers 
    
    


    Getting streamers from the Category: Grand Theft Auto V:   0%|          | 0/409 [00:00<?, ?it/s]


    Scrolling... (could take some time)
    
    Done Scrolling for streamers 
    
    


    Getting streamers from the Category: Minecraft:   0%|          | 0/55 [00:00<?, ?it/s]


    Scrolling... (could take some time)
    
    Done Scrolling for streamers 
    
    


    Getting streamers from the Category: FIFA 23:   0%|          | 0/160 [00:00<?, ?it/s]


    Scrolling... (could take some time)
    
    Done Scrolling for streamers 
    
    


    Getting streamers from the Category: VALORANT:   0%|          | 0/178 [00:00<?, ?it/s]


    Scrolling... (could take some time)
    
    Done Scrolling for streamers 
    
    


    Getting streamers from the Category: Among Us:   0%|          | 0/55 [00:00<?, ?it/s]


    Scrolling... (could take some time)
    
    Done Scrolling for streamers 
    
    


    Getting streamers from the Category: World of Warcraft:   0%|          | 0/130 [00:00<?, ?it/s]


    Scrolling... (could take some time)
    
    Done Scrolling for streamers 
    
    


    Getting streamers from the Category: Dota 2:   0%|          | 0/82 [00:00<?, ?it/s]


    Scrolling... (could take some time)
    
    Done Scrolling for streamers 
    
    


    Getting streamers from the Category: Pokémon Community Game:   0%|          | 0/55 [00:00<?, ?it/s]


    Scrolling... (could take some time)
    
    Done Scrolling for streamers 
    
    


    Getting streamers from the Category: Fortnite:   0%|          | 0/103 [00:00<?, ?it/s]


    Scrolling... (could take some time)
    
    Done Scrolling for streamers 
    
    


    Getting streamers from the Category: Apex Legends:   0%|          | 0/105 [00:00<?, ?it/s]


    Scrolling... (could take some time)
    
    Done Scrolling for streamers 
    
    


    Getting streamers from the Category: Slots:   0%|          | 0/81 [00:00<?, ?it/s]


    Scrolling... (could take some time)
    
    Done Scrolling for streamers 
    
    


    Getting streamers from the Category: Call of Duty: Warzone:   0%|          | 0/106 [00:00<?, ?it/s]


    Scrolling... (could take some time)
    
    Done Scrolling for streamers 
    
    


    Getting streamers from the Category: Counter-Strike: Global Offensive:   0%|          | 0/106 [00:00<?, ?it/s]


    Scrolling... (could take some time)
    
    Done Scrolling for streamers 
    
    


    Getting streamers from the Category: Dead by Daylight:   0%|          | 0/107 [00:00<?, ?it/s]


    Scrolling... (could take some time)
    
    Done Scrolling for streamers 
    
    


    Getting streamers from the Category: Hunt: Showdown:   0%|          | 0/56 [00:00<?, ?it/s]


    Scrolling... (could take some time)
    
    Done Scrolling for streamers 
    
    


    Getting streamers from the Category: Clash of Clans:   0%|          | 0/23 [00:00<?, ?it/s]


    Scrolling... (could take some time)
    
    Done Scrolling for streamers 
    
    


    Getting streamers from the Category: Terraria:   0%|          | 0/52 [00:00<?, ?it/s]


    Scrolling... (could take some time)
    
    Done Scrolling for streamers 
    
    


    Getting streamers from the Category: Virtual Casino:   0%|          | 0/53 [00:00<?, ?it/s]


    Scrolling... (could take some time)
    
    Done Scrolling for streamers 
    
    


    Getting streamers from the Category: Music:   0%|          | 0/53 [00:00<?, ?it/s]


    Scrolling... (could take some time)
    
    Done Scrolling for streamers 
    
    


    Getting streamers from the Category: Grounded:   0%|          | 0/55 [00:00<?, ?it/s]


    Scrolling... (could take some time)
    
    Done Scrolling for streamers 
    
    


    Getting streamers from the Category: Sports:   0%|          | 0/56 [00:00<?, ?it/s]


    Scrolling... (could take some time)
    
    Done Scrolling for streamers 
    
    


    Getting streamers from the Category: Rust:   0%|          | 0/56 [00:00<?, ?it/s]


    Scrolling... (could take some time)
    
    Done Scrolling for streamers 
    
    


    Getting streamers from the Category: Tower of Fantasy:   0%|          | 0/55 [00:00<?, ?it/s]


    Scrolling... (could take some time)
    
    Done Scrolling for streamers 
    
    


    Getting streamers from the Category: Omega Strikers:   0%|          | 0/50 [00:00<?, ?it/s]


    Scrolling... (could take some time)
    
    Done Scrolling for streamers 
    
    


    Getting streamers from the Category: The Cycle: Frontier:   0%|          | 0/55 [00:00<?, ?it/s]


    Scrolling... (could take some time)
    
    Done Scrolling for streamers 
    
    


    Getting streamers from the Category: Fear Therapy:   0%|          | 0/7 [00:00<?, ?it/s]


    Scrolling... (could take some time)
    
    Done Scrolling for streamers 
    
    


    Getting streamers from the Category: Hearthstone:   0%|          | 0/56 [00:00<?, ?it/s]

