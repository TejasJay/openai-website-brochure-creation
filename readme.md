### Detailed Explanation of the Code

This Python code defines a `Website` class designed for web scraping and using OpenAI's API to process web content into structured information and brochures. Below is a breakdown of each section:

* * *

### 1\. **Initialization (`__init__`)**

```python
def __init__(self, url):
    self.url = url
    self.body = None
    self.title = None
    self.text = None
    self.links = None
```

-   **Purpose**: The constructor initializes the class with the website's URL and placeholders for other attributes (`body`, `title`, `text`, `links`).
-   **Attributes**:
    -   `url`: Stores the website's URL.
    -   `body`: Stores the HTML content of the webpage after it's fetched.
    -   `title`: Stores the title of the webpage (extracted from `<title>` tags).
    -   `text`: Stores the cleaned textual content of the webpage (excluding scripts, styles, etc.).
    -   `links`: Stores a list of all links (`<a>` tags) found on the webpage.
* * *

### 2\. **Scraping the Website (`scrape_url`)**

```python
def scrape_url(self):
    response = requests.get(self.url, headers=headers)
    self.body = response.content
    soup = BeautifulSoup(self.body, 'html.parser')
    self.title = soup.title.text if soup.title else "No Title"
    for not_needed in soup.body(["script", "style", "img", "input"]):
        not_needed.decompose()
    self.text = soup.body.get_text(separator='\n', strip=True)
    self.links = [link.get('href') for link in soup.find_all('a')]
```

-   **Purpose**: This method fetches and processes the webpage content.
-   **Steps**:
    1.  Sends a GET request to the URL and retrieves the response.
    2.  Parses the HTML using `BeautifulSoup`.
    3.  Extracts the `<title>` text for the page title.
    4.  Removes unnecessary elements (`script`, `style`, `img`, `input`) from the HTML to clean the text.
    5.  Extracts plain text content from the `<body>` and stores it.
    6.  Collects all hyperlinks (`<a href>`) from the page and stores them in `self.links`.
* * *

### 3\. **Getting the Page Content (`get_content`)**

```python
def get_content(self):
    self.scrape_url()
    return f"Website title: {self.title}, Website content:{self.text}"
```

-   **Purpose**: Fetches and returns the title and textual content of the website.
-   **How It Works**:
    -   Calls `scrape_url()` to ensure the page content is fetched and processed.
    -   Formats and returns the title and text.
* * *

### 4\. **Generating Prompts for Link Filtering**

#### `get_link_system_prompt`

```python
def get_link_system_prompt(self):
    link_system_prompt = "You are an agent to reform a list of link provided to you from a website ..."
    return link_system_prompt
```

-   **Purpose**: Provides a system-level prompt for OpenAI. This defines the AI's role and the expected JSON structure for filtering links.
-   **Content**:
    -   Explains the task: Identify relevant links for a brochure (e.g., About Page, Careers Page).
    -   Provides an example JSON format for the response.

#### `get_link_user_prompt`

```python
def get_link_user_prompt(self):
    self.get_content()
    link_user_prompt = f"Here is the list of links on the website of {self.url} ..."
    link_user_prompt += "\n".join(self.links)
    return link_user_prompt
```

-   **Purpose**: Constructs a user-level prompt by including the website's URL and the list of scraped links.
-   **How It Works**:
    -   Calls `get_content()` to ensure content is available.
    -   Formats the links into a readable prompt.
* * *

### 5\. **Filtering Links (`get_links`)**

```python
def get_links(self):
    response = openai.chat.completions.create(
        model=MODEL,
        messages=[
            {'role': 'system', 'content': self.get_link_system_prompt()},
            {'role': 'user', 'content': self.get_link_user_prompt()}
        ],
        response_format={'type': 'json_object'}
    )
    result = response.choices[0].message.content
    link_result = json.loads(result)
    return link_result
```

-   **Purpose**: Uses OpenAI to filter the links and return relevant ones in JSON format.
-   **Steps**:
    1.  Sends system and user prompts to OpenAI.
    2.  Receives a JSON response with filtered links.
    3.  Parses and returns the response.
* * *

### 6\. **Compiling All Content (`get_all_content`)**

```python
def get_all_content(self):
    result = f"found links: {self.get_links()} \n"
    result += "Landing Page:\n"
    result += f"{self.get_content()}"
    return result
```

-   **Purpose**: Compiles all relevant data (filtered links and page content).
-   **How It Works**:
    -   Calls `get_links()` to get the filtered links.
    -   Calls `get_content()` to fetch the page content.
    -   Combines both into a formatted string.
* * *

### 7\. **Generating Prompts for Brochure Creation**

#### `get_brochure_system_prompt`

```python
def get_brochure_system_prompt(self):
    navigate_to_link_system_prompt = "You are an assistant that analyzes the contents ..."
    return navigate_to_link_system_prompt
```

-   **Purpose**: Provides a system-level prompt for creating a company brochure.
-   **Content**:
    -   Explains the AI's role (e.g., summarize company details for a brochure).
    -   Specifies what information to include (e.g., company culture, customers, careers).

#### `get_brochure_user_prompt`

```python
def get_brochure_user_prompt(self, company_name):
    navigate_to_link_user_prompt = f"You are looking at a company called: {company_name}\n"
    navigate_to_link_user_prompt += f"Here are the contents of its landing page and other relevant pages ..."
    navigate_to_link_user_prompt = navigate_to_link_user_prompt[:20_000]
    return navigate_to_link_user_prompt
```

-   **Purpose**: Constructs a user-level prompt for brochure creation.
-   **How It Works**:
    -   Includes the company name and content from `get_all_content()`.
    -   Ensures the prompt length does not exceed OpenAI's character limit.
* * *

### 8\. **Creating the Brochure (`create_brochure`)**

```python
def create_brochure(self, company_name):
    response = openai.chat.completions.create(
        model=MODEL,
        messages=[
            {"role": "system", "content": self.get_brochure_system_prompt()},
            {"role": "user", "content": self.get_brochure_user_prompt(company_name)}
        ],
    )
    result = response.choices[0].message.content
    display(Markdown(result))
```

-   **Purpose**: Uses OpenAI to generate a markdown-formatted brochure.
-   **Steps**:
    1.  Sends system and user prompts to OpenAI.
    2.  Receives and displays the AI's markdown response.
* * *

### Overall Workflow

1.  **Scraping**: Fetch and clean webpage content using `scrape_url`.
2.  **Link Filtering**: Use prompts (`get_link_system_prompt` & `get_link_user_prompt`) and OpenAI to identify relevant links.
3.  **Content Compilation**: Combine filtered links and content (`get_all_content`).
4.  **Brochure Creation**: Use prompts (`get_brochure_system_prompt` & `get_brochure_user_prompt`) and OpenAI to create a brochure.
* * *

### Key Features

1.  **Dynamic Prompt Creation**: Prompts are constructed dynamically based on the webpage content.
2.  **AI Integration**: Uses OpenAI for complex tasks like filtering links and generating brochures.
3.  **Web Scraping**: Extracts and processes webpage data effectively using `BeautifulSoup`.

This structure is modular, reusable, and highly adaptable for similar applications.
