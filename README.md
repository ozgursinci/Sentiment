import datetime
import pandas as pd
import warnings
import numpy as np
from transformers import AutoTokenizer, AutoModelForSequenceClassification
import torch
import re
import os

# Environment setup
os.environ['CURL_CA_BUNDLE'] = ''
warnings.filterwarnings("ignore")

# Load tokenizer and model with specified proxies
proxies = {"https": "fnyproxy.fnylocal.com:8080", "http": "fnyproxy.fnylocal.com:8080"}
tokenizer = AutoTokenizer.from_pretrained("ProsusAI/finbert", proxies=proxies)
model = AutoModelForSequenceClassification.from_pretrained("ProsusAI/finbert", proxies=proxies)
torch.set_num_threads(6)

# Sentiment calculation function
def calculateSentiment(text):
    inputs = tokenizer([text], padding=True, truncation=True, return_tensors='pt')
    outputs = model(**inputs)
    predictions = torch.nn.functional.softmax(outputs.logits, dim=-1)
    positive = predictions[:, 0].tolist()
    negative = predictions[:, 1].tolist()
    neutral = predictions[:, 2].tolist()
    return positive[0], negative[0], neutral[0]

# Sentence splitting function
alphabets = "([A-Za-z])"
prefixes = "(Mr|St|Mrs|Ms|Dr)[.]"
suffixes = "(Inc|Ltd|Jr|Sr|Co)"
starters = "(Mr|Mrs|Ms|Dr|He\\s|She\\s|It\\s|They\\s|Their\\s|Our\\s|We\\s|But\\s|However\\s|That\\s|This\\s|Wherever)"
acronyms = "([A-Z][.][A-Z][.](?:[A-Z][.])?)"
websites = "[.](com|net|org|io|gov)"
digits = "([0-9])"

def split_into_sentences(text):
    text = " " + text + "  "
    text = text.replace("\n", " ")
    text = re.sub(prefixes, "\\1<prd>", text)
    text = re.sub(websites, "<prd>\\1", text)
    text = re.sub(digits + "[.]" + digits, "\\1<prd>\\2", text)
    if "..." in text:
        text = text.replace("...", "<prd><prd><prd>")
    if "Ph.D" in text:
        text = text.replace("Ph.D.", "Ph<prd>D<prd>")
    text = re.sub("\\s" + alphabets + "[.] ", " \\1<prd> ", text)
    text = re.sub(acronyms + " " + starters, "\\1<stop> \\2", text)
    text = re.sub(alphabets + "[.]" + alphabets + "[.]" + alphabets + "[.]", "\\1<prd>\\2<prd>\\3<prd>", text)
    text = re.sub(alphabets + "[.]" + alphabets + "[.]", "\\1<prd>\\2<prd>", text)
    text = re.sub(" " + suffixes + "[.] " + starters, " \\1<stop> \\2", text)
    text = re.sub(" " + suffixes + "[.]", " \\1<prd>", text)
    text = re.sub(" " + alphabets + "[.]", " \\1<prd>", text)
    text = text.replace(".", ".<stop>")
    text = text.replace("?", "?<stop>")
    text = text.replace("!", "!<stop>")
    text = text.replace("<prd>", ".")
    sentences = text.split("<stop>")
    sentences = [s.strip().upper() for s in sentences[:-1]]
    return sentences

# Data processing
newsCollection = pd.DataFrame()  # Assume this is your data loaded here
newsCollection['tickers'], newsCollection['positive'], newsCollection['negative'], newsCollection['neutral'] = np.nan, np.nan, np.nan, np.nan

# Updating sentiment analysis for each row
for index, row in newsCollection.iterrows():
    sentenceList = split_into_sentences(row['alltext'])
    tickers_found = []
    pos_scores = []
    neg_scores = []
    neu_scores = []
    
    for sentence in sentenceList:
        for ticker in usTickers:
            if ticker in sentence:
                sentiment = calculateSentiment(sentence)
                tickers_found.append(ticker)
                pos_scores.append(sentiment[0])
                neg_scores.append(sentiment[1])
                neu_scores.append(sentiment[2])
    
    newsCollection.at[index, 'tickers'] = tickers_found
    newsCollection.at[index, 'positive'] = pos_scores
    newsCollection.at[index, 'negative'] = neg_scores
    newsCollection.at[index, 'neutral'] = neu_scores

print(newsCollection[['date', 'tickers', 'positive', 'negative', 'neutral']])
