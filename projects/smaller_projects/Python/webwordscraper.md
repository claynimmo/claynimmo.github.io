---
layout: default
title: Portfolio |  Web Word Scraper
---

[Home](../../../../index.md) / [Smaller Projects](../index.md) /

# Web Word Scraper

This program uses beautiful soup and word cloud to generate a word cloud of all text that appears in a given website. The results are also output to csv for later processing.
```python
import requests
from bs4 import BeautifulSoup
from bs4.element import Comment
from collections import Counter
from urllib.parse import urlparse
from wordcloud import WordCloud
import matplotlib.pyplot as plt
import csv
import os


def is_valid_url(url: str) -> bool:
    try:
        result = urlparse(url)
        return all([result.scheme, result.netloc])
    except:
        return False

def get_url(prompt):
    """Prompt the user to get the url of the website"""
    while True:
        value = input(prompt).strip()
        if is_valid_url(value):
            return value
        print("Input cannot be empty. Please try again.")

def is_visible(element):
    if element.parent.name in ["style", "script", "head", "title", "meta", "[document]"]:
        return False
    if isinstance(element, Comment):
        return False
    return True

def write_to_file(data, title):
    """
    output the data into a usable csv file
    title: title of the webpage, to be used as the file name
    data: dictionary representing the words mapped to the amount of times it appeared
    """
    scriptDir = os.path.dirname(os.path.abspath(__file__))
    fileName = title.replace(" ", "_") + ".csv"
    fileName = os.path.join(scriptDir, fileName)
    with open(fileName, "w", newline="", encoding="utf-8") as f:
        writer = csv.writer(f)
        writer.writerow(["word", "count"])
    
        for word, freq in data.items():
            writer.writerow([word, freq])
    print(f"Wrote to file: {fileName}")

def scrape_website(url):
    """
    scrape the website to get the word frequency, printing it to the terminal, outputing to csv, and displaying the results in a word cloud
    url: string url to the website
    """
    if(is_valid_url(url) == False): #recheck for if the url is valid, just in case
        return
    
    response = requests.get(url)
    soup = BeautifulSoup(response.text, "html.parser")

    #extract features from common html elements that contain text
    headings = [h.get_text(strip=True) for h in soup.find_all(["h1","h2","h3","h4","h5","h6"])]
    paragraphs = [p.get_text(strip=True) for p in soup.find_all("p")]
    links = [a.get_text(strip=True) for a in soup.find_all("a")]

    #get the full text as one large list, and combined into a single string to use in the wordcloud
    full_text = headings + paragraphs + links
    full_text_str = " ".join(full_text)

    wc = WordCloud(width=800, height=400).generate(full_text_str)

    freq = wc.words_

    top_words = sorted(freq.items(), key=lambda x: x[1], reverse=True)

    #since the word cloud gives normalized results, multiply by the amount of words to get the real value
    unnormalized_top_words = {k: v * len(full_text) for k, v in top_words}

    title = soup.title.string if soup.title else "default_title"

    #print to the terminal
    for i, (k, v) in enumerate(unnormalized_top_words.items()):
        print(f"{i}: {k}, appeared {v} times")

    #output to file
    write_to_file(unnormalized_top_words, title)

    plt.imshow(wc, interpolation="bilinear")
    plt.axis("off")
    plt.show()


if __name__ == "__main__":
    url = get_url("Please Input URL: ")
    scrape_website(url)
```