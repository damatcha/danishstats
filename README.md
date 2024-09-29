## Overview
This project is a web-based Python tool for automating the search of Danish genealogy and historical data across multiple databases: GENI, DST, SA (Scanned Archives), and DAISY. It fetches the results by scraping these websites and aggregates the output in an Excel file.

## Features

- **Automated search**: The tool automates searching for names in multiple databases and extracting relevant results.
- **Web scraping**: It scrapes data using the `BeautifulSoup` library to extract specific information from HTML pages.
- **Excel file processing**: Input names from an Excel file and outputs search results (URLs and counts) into a new Excel file.
- **Flask-based web interface**: The application runs a Flask web server where users can trigger the search and monitor its progress.
- **Batch processing**: Names are processed in batches, and there is a delay between batch requests to avoid overwhelming the servers.
- **Logging**: Logs are generated to track the progress of the search.

## Requirements

The following Python packages are required for this tool:
- `Flask`
- `pandas`
- `requests`
- `openpyxl`
- `beautifulsoup4`
- `re`
- `threading`

To install all the required packages, you can run:

```bash
pip install Flask pandas requests openpyxl beautifulsoup4
```

## How It Works

### 1. Input File

The tool accepts an Excel file with a column named `NAMES` containing the names to be searched. 

Example input file (`names_list.xlsx`):

| NAMES  |
|--------|
| John   |
| Alice  |
| Emma   |

### 2. Search Process

- For each name, the tool searches four databases: `GENI`, `DST`, `SA`, and `DAISY`.
- The number of search results and the URL for the search are extracted from each database.
- Results are saved in the Excel output file in corresponding columns: `GENI URL`, `GENI COUNT`, `DST URL`, `DST COUNT`, etc.

### 3. Output File

The tool generates an Excel output file (`results_denmark_statistics.xlsx`) containing:

- URLs where results were found
- The number of results per database

Output file example:

| NAMES | GENI URL    | GENI COUNT | DST URL    | DST COUNT | SA URL    | SA COUNT | DAISY URL  | DAISY COUNT |
|-------|-------------|------------|------------|-----------|-----------|----------|------------|-------------|
| John  | Geni Link 1 | 5          | DST Link 1 | 10        | SA Link 1 | 3        | Daisy Link | 12          |
| Alice | Geni Link 2 | 8          | DST Link 2 | 0         | SA Link 2 | 1        | Daisy Link | 0           |

## Running the Tool

1. **Start the server**:
   When running the script, a Flask server is started on `http://127.0.0.1:XXXX`. Replace `XXXX` with the desired port number.

   ```bash
   python script.py
   ```

2. **Access the web interface**:
   After starting the server, your web browser will automatically open a page to trigger the process (`/auth` endpoint). The interface displays a message when processing starts.

3. **Provide the Excel file**:
   After triggering the process, enter the path of the Excel file when prompted.

4. **Processing**:
   The tool will begin searching through each database and writing the results to an output Excel file. Processing logs will appear in the terminal.

## Flask Endpoints

- `/auth`: Simple authentication endpoint to start the process.
- `/process`: Starts the background thread that processes the Excel file.

## Configuration

- `REQUEST_DELAY`: A delay (in seconds) between batch requests to avoid overwhelming the search databases.
- `BASE_URLS`: Dictionary containing the base URLs of the different databases.
- `URL_SUFFIXES`: Dictionary containing the suffixes for constructing database search URLs.
- `OUTPUT_FILE`: The name of the output Excel file.

## Example Usage

- You are given an Excel file with a list of names to search.
- The tool will search for each name across the four Danish databases (GENI, DST, SA, and DAISY).
- The search results are compiled into a new Excel file with URLs and counts for each database.

## Error Handling

- The tool handles failed search requests by logging errors and continuing the search for the next name.
- It validates that the input Excel file exists and that the expected columns are present.
- If a search result page doesn't contain the expected structure (e.g., result count), the tool assigns a count of `0`.

## Logging

The script uses Python's built-in logging system to log information about the process, such as:
- Start and completion of each name's search
- Errors encountered during the search or Excel file processing
- Success of saving the output Excel file

Logs are printed to the console.

## To-Do

- Add authentication support if needed for protected resources.
- Improve the robustness of HTML parsing for sites that change frequently.
- Add support for parallel searches to optimize runtime.

## License

This project is licensed under the MIT License.
