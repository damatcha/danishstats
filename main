import os
import time
import pandas as pd
import requests
from flask import Flask, redirect
import threading
import webbrowser
from bs4 import BeautifulSoup
import logging
import re
from typing import Optional, Tuple

REQUEST_DELAY = 30 
BASE_URLS = {
    'GENI': 'https://www.geni.com/search?search_type=people&names=',
    'DST': 'https://www.dst.dk/da/search?q=',
    'SA': 'https://www.sa.dk/find/#q=',
    'DAISY': 'https://daisy.rigsarkivet.dk/arkivskaber_eller_arkivserie_liste?c='
}
URL_SUFFIXES = {
    'GENI': '',
    'DST': '&ui=dstdk',
    'SA': '&side=1&at=Scannede+arkivalier',
    'DAISY': '&d=1&e=2024'
}
OUTPUT_FILE = 'results_denmark_statistics.xlsx'

app = Flask(__name__)

excel_file_path = None

logging.basicConfig(level=logging.INFO)

@app.route('/auth')
def auth():
    return redirect('/process')

@app.route('/process')
def process():
    threading.Thread(target=process_excel_file).start()
    return 'Processing started! You can close this window.'

def extract_result_count(response_text: str, db_name: str) -> int:
    if db_name == 'SA':
        return extract_number_from_sa(response_text)
    elif db_name == 'DAISY':
        return extract_number_from_daisy(response_text)
    else:
        soup = BeautifulSoup(response_text, 'html.parser')
        
        if db_name == 'GENI':
            count = sum('Danmark' in div.get_text() for div in soup.find_all('div', class_='small'))
        elif db_name == 'DST':
            # Extract number of results from the lead text
            lead_text = soup.find('p', class_='lead')
            if lead_text:
                text = lead_text.text
                if "Prøv et andet søgeord" in text:
                    count = 0
                elif 'af' in text:
                    try:
                        count = int(text.split('af')[-1].strip())
                    except ValueError:
                        count = 0
                else:
                    count = 0
            else:
                count = 0
        else:
            count = 0
        
        return count

SA_DIV_CLASSES = ['span2 nr-of-posts', 'span2 floating-pagination']

def extract_text_from_div(response_text: str, div_class: str) -> Optional[str]:
    soup = BeautifulSoup(response_text, 'html.parser')
    div = soup.find('div', class_=div_class)
    if div:
        return div.get_text(strip=True)
    else:
        return None

def extract_number_from_text(text: str) -> Optional[int]:
    text = text.strip()

    pattern = r'af (\d+) poster'
    match = re.search(pattern, text)
    if match:
        number_text = match.group(1)
        return int(number_text)
    else:
        return None

def extract_number_from_sa(response_text: str) -> Optional[int]:
    for div_class in SA_DIV_CLASSES:
        text = extract_text_from_div(response_text, div_class)
        if text:
            number = extract_number_from_text(text)
            if number is not None:
                return number

    return 0

def extract_number_from_daisy(response_text: str) -> Optional[int]:
    soup = BeautifulSoup(response_text, 'html.parser')
    result_div = soup.find('div', string=lambda x: x and 'Der blev fundet' in x)
    if result_div:
        b_tag = result_div.find('b')
        if b_tag:
            try:
                count = int(b_tag.get_text())
            except ValueError:
                count = 0
        else:
            count = 0
    else:
        count = 0
    return count

def search_database(base_url: str, name: str, suffix: str = '') -> Tuple[Optional[str], str]:
    try:
        search_url = f"{base_url}{name}{suffix}"
        response = requests.get(search_url)
        response.raise_for_status()
        return search_url, response.text
    except requests.RequestException as e:
        logging.error(f"Error searching database for name '{name}': {e}")
        return None, ''

def search_geni(name: str) -> Tuple[Optional[str], int]:
    total_count = 0
    page_number = 1
    base_url = BASE_URLS['GENI']
    
    while True:
        suffix = f'&page={page_number}' 
        url, response_text = search_database(base_url, name, suffix)
        
        if url is None:
            break
        
        soup = BeautifulSoup(response_text, 'html.parser')
        current_page_count = extract_result_count(response_text, 'GENI')
        
        if current_page_count == 0:
            break
        
        total_count += current_page_count
        page_number += 1  # Move to the next page

    return base_url, total_count

def process_excel(file_path: str):
    try:
        df = pd.read_excel(file_path, engine='openpyxl')
        logging.info("Columns in the Excel file: %s", df.columns)
    except Exception as e:
        logging.error("Error reading Excel file: %s", e)
        return

    name_column = 'NAMES'
    url_columns = {
        'GENI': 'GENI URL',
        'DST': 'DST URL',
        'SA': 'SA URL',
        'DAISY': 'DAISY URL'
    }
    count_columns = {
        'GENI': 'GENI COUNT',
        'DST': 'DST COUNT',
        'SA': 'SA COUNT',
        'DAISY': 'DAISY COUNT'
    }

    if name_column not in df.columns:
        logging.error("Error: '%s' column not found in the Excel file.", name_column)
        return
  
    for column in list(url_columns.values()) + list(count_columns.values()):
        if column not in df.columns:
            df[column] = ''

    all_names = df[name_column].dropna().tolist()

    for i in range(0, len(all_names), 10):
        names_batch = all_names[i:i + 10]
        logging.info("Processing batch %d: Names %d to %d", i // 10 + 1, i + 1, i + len(names_batch))

        for name in names_batch:
            logging.info("Searching for '%s' in GENI, DST, SA, and DAISY databases...", name)
            
            results = {}
            for db_name, base_url in BASE_URLS.items():
                suffix = URL_SUFFIXES[db_name]
                if db_name == 'GENI':
                    url, count = search_geni(name)
                    results[db_name] = (url, count)
                else:
                    url, response = search_database(base_url, name, suffix)
                    if url is None:
                        results[db_name] = (None, 0)
                    else:
                        number = extract_result_count(response, db_name)
                        results[db_name] = (url, number)

            for db_name in BASE_URLS.keys():
                df.loc[df[name_column] == name, url_columns[db_name]] = results[db_name][0]
                df.loc[df[name_column] == name, count_columns[db_name]] = results[db_name][1]

        time.sleep(REQUEST_DELAY)  # Delay between batches

    df.to_excel(OUTPUT_FILE, index=False)
    logging.info("Results saved to %s", OUTPUT_FILE)

def process_excel_file():
    global excel_file_path

    if os.path.isfile(excel_file_path):
        logging.info("Processing file: %s", excel_file_path)
        process_excel(excel_file_path)
    else:
        logging.error("Excel file not found. Exiting.")

def main():
    global excel_file_path
    excel_file_path = input("Enter the path to the Excel file: ")

    # Start Flask server in a separate thread
    def run_server():
        logging.info("Starting Flask server on http://127.0.0.1:XXXX")
        app.run(host='127.0.0.1', port=XXXX)

    server_thread = threading.Thread(target=run_server)
    server_thread.start()


    webbrowser.open('http://127.0.0.1:XXXX/auth')

    input("Press Enter once you have authenticated...")

    if os.path.isfile(excel_file_path):
        process_excel_file()
    else:
        logging.error("Excel file not found. Exiting.")

if __name__ == '__main__':
    main()
