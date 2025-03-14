# Serper API Documentation

```
{
    "openapi": "3.0.0",
    "info": {
        "title": "SerperDev",
        "version": "1.0.0",
        "description": "API for performing search queries"
    },
    "servers": [
        {
            "url": "https://google.serper.dev"
        }
    ],
    "paths": {
        "/search": {
            "post": {
                "operationId": "search",
                "description": "Search the web with Google",
                "requestBody": {
                    "required": true,
                    "content": {
                        "application/json": {
                            "schema": {
                                "type": "object",
                                "properties": {
                                    "q": {
                                        "type": "string"
                                    }
                                }
                            }
                        }
                    }
                },
                "responses": {
                    "200": {
                        "description": "Successful response",
                        "content": {
                            "application/json": {
                                "schema": {
                                    "type": "object",
                                    "properties": {
                                        "searchParameters": {
                                            "type": "undefined"
                                        },
                                        "knowledgeGraph": {
                                            "type": "undefined"
                                        },
                                        "answerBox": {
                                            "type": "undefined"
                                        },
                                        "organic": {
                                            "type": "undefined"
                                        },
                                        "topStories": {
                                            "type": "undefined"
                                        },
                                        "peopleAlsoAsk": {
                                            "type": "undefined"
                                        },
                                        "relatedSearches": {
                                            "type": "undefined"
                                        }
                                    }
                                }
                            }
                        }
                    }
                },
                "security": [
                    {
                        "apikey": []
                    }
                ]
            }
        }
    },
    "components": {
        "securitySchemes": {
            "apikey": {
                "type": "apiKey",
                "name": "x-api-key",
                "in": "header"
            }
        }
    }
}
```

# Firecrawl API Documentation

## Features

**Scrape**

Extract content from any webpage in markdown or json format.

**Crawl**

Crawl entire websites, extract their content and metadata.

**Map**

Get a complete list of URLs from any website quickly and reliably.

**Extract**

Extract structured data from entire webpages using natural language.

**Search**

Search the web and get full page content in any format.

## Base URL

All requests contain the following base URL:

```
https://api.firecrawl.dev
```

## Authentication

For authentication, it’s required to include an Authorization header. The header should contain `Bearer fc-123456789`, where `fc-123456789` represents your API Key.

```
Authorization: Bearer fc-123456789
```

## Response codes

Firecrawl employs conventional HTTP status codes to signify the outcome of your requests.

Typically, 2xx HTTP status codes denote success, 4xx codes represent failures related to the user, and 5xx codes signal infrastructure problems.

| Status | Description |
| --- | --- |
| 200 | Request was successful. |
| 400 | Verify the correctness of the parameters. |
| 401 | The API key was not provided. |
| 402 | Payment required |
| 404 | The requested resource could not be located. |
| 429 | The rate limit has been surpassed. |
| 5xx | Signifies a server error with Firecrawl. |

Refer to the Error Codes section for a detailed explanation of all potential API errors.

## Rate limit

The Firecrawl API has a rate limit to ensure the stability and reliability of the service. The rate limit is applied to all endpoints and is based on the number of requests made within a specific time frame.

When you exceed the rate limit, you will receive a 429 response code.

## Scrape Endpoint

POST

/scrape

```
curl --request POST \
  --url https://api.firecrawl.dev/v1/scrape \
  --header 'Authorization: Bearer <token>' \
  --header 'Content-Type: application/json' \
  --data '{
  "url": "<string>",
  "formats": [\
    "markdown"\
  ],
  "onlyMainContent": true,
  "includeTags": [\
    "<string>"\
  ],
  "excludeTags": [\
    "<string>"\
  ],
  "headers": {},
  "waitFor": 0,
  "mobile": false,
  "skipTlsVerification": false,
  "timeout": 30000,
  "jsonOptions": {
    "schema": {},
    "systemPrompt": "<string>",
    "prompt": "<string>"
  },
  "actions": [\
    {\
      "type": "wait",\
      "milliseconds": 2,\
      "selector": "#my-element"\
    }\
  ],
  "location": {
    "country": "US",
    "languages": [\
      "en-US"\
    ]
  },
  "removeBase64Images": true,
  "blockAds": true,
  "proxy": "basic"
}'
```

```
{
  "success": true,
  "data": {
    "markdown": "<string>",
    "html": "<string>",
    "rawHtml": "<string>",
    "screenshot": "<string>",
    "links": [\
      "<string>"\
    ],
    "actions": {
      "screenshots": [\
        "<string>"\
      ]
    },
    "metadata": {
      "title": "<string>",
      "description": "<string>",
      "language": "<string>",
      "sourceURL": "<string>",
      "<any other metadata> ": "<string>",
      "statusCode": 123,
      "error": "<string>"
    },
    "llm_extraction": {},
    "warning": "<string>"
  }
}
```

### Authorizations

Authorization

string

header

required

Bearer authentication header of the form `Bearer <token>`, where `<token>` is your auth token.

### Body

application/json

url

string

required

The URL to scrape

formats

enum<string>[]

Formats to include in the output.

Available options:

`markdown`,

`html`,

`rawHtml`,

`links`,

`screenshot`,

`screenshot@fullPage`,

`json`

onlyMainContent

boolean

default:true

Only return the main content of the page excluding headers, navs, footers, etc.

includeTags

string[]

Tags to include in the output.

excludeTags

string[]

Tags to exclude from the output.

headers

object

Headers to send with the request. Can be used to send cookies, user-agent, etc.

waitFor

integer

default:0

Specify a delay in milliseconds before fetching the content, allowing the page sufficient time to load.

mobile

boolean

default:false

Set to true if you want to emulate scraping from a mobile device. Useful for testing responsive pages and taking mobile screenshots.

skipTlsVerification

boolean

default:false

Skip TLS certificate verification when making requests

timeout

integer

default:30000

Timeout in milliseconds for the request

jsonOptions

object

Extract object

schema

object

The schema to use for the extraction (Optional)

jsonOptions.systemPrompt

string

The system prompt to use for the extraction (Optional)

jsonOptions.prompt

string

The prompt to use for the extraction without a schema (Optional)

actions

object[]

Actions to perform on the page before grabbing the content

* Wait
* Screenshot
* Click
* Write text
* Press a key
* Scroll
* Scrape
* Execute JavaScript

actions.type

enum<string>

required

Wait for a specified amount of milliseconds

Available options:

`wait`

actions.milliseconds

integer

Number of milliseconds to wait

Required range: `x >= 1`

actions.selector

string

Query selector to find the element by

Example:

`"#my-element"`

location

object

Location settings for the request. When specified, this will use an appropriate proxy if available and emulate the corresponding language and timezone settings. Defaults to 'US' if not specified.

location.country

string

default:US

ISO 3166-1 alpha-2 country code (e.g., 'US', 'AU', 'DE', 'JP')

location.languages

string[]

Preferred languages and locales for the request in order of priority. Defaults to the language of the specified location. See https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept-Language

removeBase64Images

boolean

Removes all base 64 images from the output, which may be overwhelmingly long. The image's alt text remains in the output, but the URL is replaced with a placeholder.

blockAds

boolean

default:true

Enables ad-blocking and cookie popup blocking.

proxy

enum<string>

Specifies the type of proxy to use.

* basic: Proxies for scraping sites with none to basic anti-bot solutions. Fast and usually works.
* stealth: Stealth proxies for scraping sites with advanced anti-bot solutions. Slower, but more reliable on certain sites.

If you do not specify a proxy, Firecrawl will automatically attempt to determine which one you need based on the target site.

Available options:

`basic`,

`stealth`

### Response

200

application/json

Successful response

success

boolean

data

object

markdown

string

data.html

string | null

HTML version of the content on page if `html` is in `formats`

data.rawHtml

string | null

Raw HTML content of the page if `rawHtml` is in `formats`

data.screenshot

string | null

Screenshot of the page if `screenshot` is in `formats`

data.links

string[]

List of links on the page if `links` is in `formats`

data.actions

object | null

Results of the actions specified in the `actions` parameter. Only present if the `actions` parameter was provided in the request

data.actions.screenshots

string[]

Screenshot URLs, in the same order as the screenshot actions provided.

data.metadata

object

data.metadata.title

string

data.metadata.description

string

data.metadata.language

string | null

data.metadata.sourceURL

string

data.metadata.<any other metadata>

string

data.metadata.statusCode

integer

data.metadata.error

string | null

data.llm_extraction

object | null

Displayed when using LLM Extraction. Extracted data from the page following the schema defined.

data.warning

string | null

Can be displayed when using LLM Extraction. Warning message will let you know any issues with the extraction.

## Crawl Endpoint

POST

/crawl

```
curl --request POST \
  --url https://api.firecrawl.dev/v1/crawl \
  --header 'Authorization: Bearer <token>' \
  --header 'Content-Type: application/json' \
  --data '{
  "url": "<string>",
  "excludePaths": [\
    "<string>"\
  ],
  "includePaths": [\
    "<string>"\
  ],
  "maxDepth": 10,
  "maxDiscoveryDepth": 123,
  "ignoreSitemap": false,
  "ignoreQueryParameters": false,
  "limit": 10000,
  "allowBackwardLinks": false,
  "allowExternalLinks": false,
  "webhook": {
    "url": "<string>",
    "headers": {},
    "metadata": {},
    "events": [\
      "completed"\
    ]
  },
  "scrapeOptions": {
    "formats": [\
      "markdown"\
    ],
    "onlyMainContent": true,
    "includeTags": [\
      "<string>"\
    ],
    "excludeTags": [\
      "<string>"\
    ],
    "headers": {},
    "waitFor": 0,
    "mobile": false,
    "skipTlsVerification": false,
    "timeout": 30000,
    "jsonOptions": {
      "schema": {},
      "systemPrompt": "<string>",
      "prompt": "<string>"
    },
    "actions": [\
      {\
        "type": "wait",\
        "milliseconds": 2,\
        "selector": "#my-element"\
      }\
    ],
    "location": {
      "country": "US",
      "languages": [\
        "en-US"\
      ]
    },
    "removeBase64Images": true,
    "blockAds": true,
    "proxy": "basic"
  }
}'
```

```
{
  "success": true,
  "id": "<string>",
  "url": "<string>"
}
```

### Authorizations

Authorization

string

header

required

Bearer authentication header of the form `Bearer <token>`, where `<token>` is your auth token.

### Body

application/json

url

string

required

The base URL to start crawling from

excludePaths

string[]

URL pathname regex patterns that exclude matching URLs from the crawl. For example, if you set "excludePaths": \["blog/.\*"\] for the base URL firecrawl.dev, any results matching that pattern will be excluded, such as https://www.firecrawl.dev/blog/firecrawl-launch-week-1-recap.

includePaths

string[]

URL pathname regex patterns that include matching URLs in the crawl. Only the paths that match the specified patterns will be included in the response. For example, if you set "includePaths": \["blog/.\*"\] for the base URL firecrawl.dev, only results matching that pattern will be included, such as https://www.firecrawl.dev/blog/firecrawl-launch-week-1-recap.

maxDepth

integer

default:10

Maximum depth to crawl relative to the base URL. Basically, the max number of slashes the pathname of a scraped URL may contain.

maxDiscoveryDepth

integer

Maximum depth to crawl based on discovery order. The root site and sitemapped pages has a discovery depth of 0. For example, if you set it to 1, and you set ignoreSitemap, you will only crawl the entered URL and all URLs that are linked on that page.

ignoreSitemap

boolean

default:false

Ignore the website sitemap when crawling

ignoreQueryParameters

boolean

default:false

Do not re-scrape the same path with different (or none) query parameters

limit

integer

default:10000

Maximum number of pages to crawl. Default limit is 10000.

allowBackwardLinks

boolean

default:false

Enables the crawler to navigate from a specific URL to previously linked pages.

allowExternalLinks

boolean

default:false

Allows the crawler to follow links to external websites.

webhook

object

A webhook specification object.

webhook.url

string

required

The URL to send the webhook to. This will trigger for crawl started (crawl.started), every page crawled (crawl.page) and when the crawl is completed (crawl.completed or crawl.failed). The response will be the same as the `/scrape` endpoint.

webhook.headers

object

Headers to send to the webhook URL.

webhook.headers.{key}

string

webhook.metadata

object

Custom metadata that will be included in all webhook payloads for this crawl

webhook.events

enum<string>[]

Type of events that should be sent to the webhook URL. (default: all)

Available options:

`completed`,

`page`,

`failed`,

`started`

scrapeOptions

object

scrapeOptions.formats

enum<string>[]

Formats to include in the output.

Available options:

`markdown`,

`html`,

`rawHtml`,

`links`,

`screenshot`,

`screenshot@fullPage`,

`json`

scrapeOptions.onlyMainContent

boolean

default:true

Only return the main content of the page excluding headers, navs, footers, etc.

scrapeOptions.includeTags

string[]

Tags to include in the output.

scrapeOptions.excludeTags

string[]

Tags to exclude from the output.

scrapeOptions.headers

object

Headers to send with the request. Can be used to send cookies, user-agent, etc.

scrapeOptions.waitFor

integer

default:0

Specify a delay in milliseconds before fetching the content, allowing the page sufficient time to load.

scrapeOptions.mobile

boolean

default:false

Set to true if you want to emulate scraping from a mobile device. Useful for testing responsive pages and taking mobile screenshots.

scrapeOptions.skipTlsVerification

boolean

default:false

Skip TLS certificate verification when making requests

scrapeOptions.timeout

integer

default:30000

Timeout in milliseconds for the request

scrapeOptions.jsonOptions

object

Extract object

scrapeOptions.jsonOptions.schema

object

The schema to use for the extraction (Optional)

scrapeOptions.jsonOptions.systemPrompt

string

The system prompt to use for the extraction (Optional)

scrapeOptions.jsonOptions.prompt

string

The prompt to use for the extraction without a schema (Optional)

scrapeOptions.actions

object[]

Actions to perform on the page before grabbing the content

* Wait
* Screenshot
* Click
* Write text
* Press a key
* Scroll
* Scrape
* Execute JavaScript

scrapeOptions.actions.type

enum<string>

required

Wait for a specified amount of milliseconds

Available options:

`wait`

scrapeOptions.actions.milliseconds

integer

Number of milliseconds to wait

Required range: `x >= 1`

scrapeOptions.actions.selector

string

Query selector to find the element by

Example:

`"#my-element"`

scrapeOptions.location

object

Location settings for the request. When specified, this will use an appropriate proxy if available and emulate the corresponding language and timezone settings. Defaults to 'US' if not specified.

scrapeOptions.location.country

string

default:US

ISO 3166-1 alpha-2 country code (e.g., 'US', 'AU', 'DE', 'JP')

scrapeOptions.location.languages

string[]

Preferred languages and locales for the request in order of priority. Defaults to the language of the specified location. See https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept-Language

scrapeOptions.removeBase64Images

boolean

Removes all base 64 images from the output, which may be overwhelmingly long. The image's alt text remains in the output, but the URL is replaced with a placeholder.

scrapeOptions.blockAds

boolean

default:true

Enables ad-blocking and cookie popup blocking.

scrapeOptions.proxy

enum<string>

Specifies the type of proxy to use.

* basic: Proxies for scraping sites with none to basic anti-bot solutions. Fast and usually works.
* stealth: Stealth proxies for scraping sites with advanced anti-bot solutions. Slower, but more reliable on certain sites.

If you do not specify a proxy, Firecrawl will automatically attempt to determine which one you need based on the target site.

Available options:

`basic`,

`stealth`

### Response

200

application/json

Successful response

success

boolean

id

string

url

string

## Map Endpoint

POST

/map

```
curl --request POST \
  --url https://api.firecrawl.dev/v1/map \
  --header 'Authorization: Bearer <token>' \
  --header 'Content-Type: application/json' \
  --data '{
  "url": "<string>",
  "search": "<string>",
  "ignoreSitemap": true,
  "sitemapOnly": false,
  "includeSubdomains": false,
  "limit": 5000,
  "timeout": 123
}'
```

```
{
  "success": true,
  "links": [\
    "<string>"\
  ]
}
```

### Authorizations

Authorization

string

header

required

Bearer authentication header of the form `Bearer <token>`, where `<token>` is your auth token.

### Body

application/json

url

string

required

The base URL to start crawling from

search

string

Search query to use for mapping. During the Alpha phase, the 'smart' part of the search functionality is limited to 1000 search results. However, if map finds more results, there is no limit applied.

ignoreSitemap

boolean

default:true

Ignore the website sitemap when crawling.

sitemapOnly

boolean

default:false

Only return links found in the website sitemap

includeSubdomains

boolean

default:false

Include subdomains of the website

limit

integer

default:5000

Maximum number of links to return

Required range: `x <= 5000`

timeout

integer

Timeout in milliseconds. There is no timeout by default.

### Response

200

application/json

Successful response

success

boolean

links

string[]

## Extract Endpoint

POST

/extract

```
curl --request POST \
  --url https://api.firecrawl.dev/v1/extract \
  --header 'Authorization: Bearer <token>' \
  --header 'Content-Type: application/json' \
  --data '{
  "urls": [\
    "<string>"\
  ],
  "prompt": "<string>",
  "schema": {
    "property1": "<string>",
    "property2": 123
  },
  "enableWebSearch": false,
  "ignoreSitemap": false,
  "includeSubdomains": true,
  "showSources": false,
  "scrapeOptions": {
    "formats": [\
      "markdown"\
    ],
    "onlyMainContent": true,
    "includeTags": [\
      "<string>"\
    ],
    "excludeTags": [\
      "<string>"\
    ],
    "headers": {},
    "waitFor": 0,
    "mobile": false,
    "skipTlsVerification": false,
    "timeout": 30000,
    "jsonOptions": {
      "schema": {},
      "systemPrompt": "<string>",
      "prompt": "<string>"
    },
    "actions": [\
      {\
        "type": "wait",\
        "milliseconds": 2,\
        "selector": "#my-element"\
      }\
    ],
    "location": {
      "country": "US",
      "languages": [\
        "en-US"\
      ]
    },
    "removeBase64Images": true,
    "blockAds": true,
    "proxy": "basic"
  }
}'
```

```
{
  "success": true,
  "id": "<string>"
}
```

### Authorizations

Authorization

string

header

required

Bearer authentication header of the form `Bearer <token>`, where `<token>` is your auth token.

### Body

application/json

urls

string[]

required

The URLs to extract data from. URLs should be in glob format.

prompt

string

Prompt to guide the extraction process

schema

object

Schema to define the structure of the extracted data

schema.property1

string

required

Description of property1

schema.property2

integer

required

Description of property2

enableWebSearch

boolean

default:false

When true, the extraction will use web search to find additional data

ignoreSitemap

boolean

default:false

When true, sitemap.xml files will be ignored during website scanning

includeSubdomains

boolean

default:true

When true, subdomains of the provided URLs will also be scanned

showSources

boolean

default:false

When true, the sources used to extract the data will be included in the response as `sources` key

scrapeOptions

object

scrapeOptions.formats

enum<string>[]

Formats to include in the output.

Available options:

`markdown`,

`html`,

`rawHtml`,

`links`,

`screenshot`,

`screenshot@fullPage`,

`json`

scrapeOptions.onlyMainContent

boolean

default:true

Only return the main content of the page excluding headers, navs, footers, etc.

scrapeOptions.includeTags

string[]

Tags to include in the output.

scrapeOptions.excludeTags

string[]

Tags to exclude from the output.

scrapeOptions.headers

object

Headers to send with the request. Can be used to send cookies, user-agent, etc.

scrapeOptions.waitFor

integer

default:0

Specify a delay in milliseconds before fetching the content, allowing the page sufficient time to load.

scrapeOptions.mobile

boolean

default:false

Set to true if you want to emulate scraping from a mobile device. Useful for testing responsive pages and taking mobile screenshots.

scrapeOptions.skipTlsVerification

boolean

default:false

Skip TLS certificate verification when making requests

scrapeOptions.timeout

integer

default:30000

Timeout in milliseconds for the request

scrapeOptions.jsonOptions

object

Extract object

scrapeOptions.jsonOptions.schema

object

The schema to use for the extraction (Optional)

scrapeOptions.jsonOptions.systemPrompt

string

The system prompt to use for the extraction (Optional)

scrapeOptions.jsonOptions.prompt

string

The prompt to use for the extraction without a schema (Optional)

scrapeOptions.actions

object[]

Actions to perform on the page before grabbing the content

* Wait
* Screenshot
* Click
* Write text
* Press a key
* Scroll
* Scrape
* Execute JavaScript

scrapeOptions.actions.type

enum<string>

required

Wait for a specified amount of milliseconds

Available options:

`wait`

scrapeOptions.actions.milliseconds

integer

Number of milliseconds to wait

Required range: `x >= 1`

scrapeOptions.actions.selector

string

Query selector to find the element by

Example:

`"#my-element"`

scrapeOptions.location

object

Location settings for the request. When specified, this will use an appropriate proxy if available and emulate the corresponding language and timezone settings. Defaults to 'US' if not specified.

scrapeOptions.location.country

string

default:US

ISO 3166-1 alpha-2 country code (e.g., 'US', 'AU', 'DE', 'JP')

scrapeOptions.location.languages

string[]

Preferred languages and locales for the request in order of priority. Defaults to the language of the specified location. See https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Accept-Language

scrapeOptions.removeBase64Images

boolean

Removes all base 64 images from the output, which may be overwhelmingly long. The image's alt text remains in the output, but the URL is replaced with a placeholder.

scrapeOptions.blockAds

boolean

default:true

Enables ad-blocking and cookie popup blocking.

scrapeOptions.proxy

enum<string>

Specifies the type of proxy to use.

* basic: Proxies for scraping sites with none to basic anti-bot solutions. Fast and usually works.
* stealth: Stealth proxies for scraping sites with advanced anti-bot solutions. Slower, but more reliable on certain sites.

If you do not specify a proxy, Firecrawl will automatically attempt to determine which one you need based on the target site.

Available options:

`basic`,

`stealth`

### Response

200

application/json

Successful extraction

success

boolean

id

string

## Search Endpoint

POST

/search

```
curl --request POST \
  --url https://api.firecrawl.dev/v1/search \
  --header 'Authorization: Bearer <token>' \
  --header 'Content-Type: application/json' \
  --data '{
  "query": "<string>",
  "limit": 5,
  "tbs": "<string>",
  "lang": "en",
  "country": "us",
  "location": "<string>",
  "timeout": 60000,
  "scrapeOptions": {}
}'
```

```
{
  "success": true,
  "data": [\
    {\
      "title": "<string>",\
      "description": "<string>",\
      "url": "<string>",\
      "markdown": "<string>",\
      "html": "<string>",\
      "rawHtml": "<string>",\
      "links": [\
        "<string>"\
      ],\
      "screenshot": "<string>",\
      "metadata": {\
        "title": "<string>",\
        "description": "<string>",\
        "sourceURL": "<string>",\
        "statusCode": 123,\
        "error": "<string>"\
      }\
    }\
  ],
  "warning": "<string>"
}
```

The search endpoint combines web search (SERP) with Firecrawl’s scraping capabilities to return full page content for any query.

Include `scrapeOptions` with `formats: ["markdown"]` to get complete markdown content for each search result otherwise you will default to getting the SERP results (url, title, description).

### Authorizations

Authorization

string

header

required

Bearer authentication header of the form `Bearer <token>`, where `<token>` is your auth token.

### Body

application/json

query

string

required

The search query

limit

integer

default:5

Maximum number of results to return

Required range: `1 <= x <= 10`

tbs

string

Time-based search parameter

lang

string

default:en

Language code for search results

country

string

default:us

Country code for search results

location

string

Location parameter for search results

timeout

integer

default:60000

Timeout in milliseconds

scrapeOptions

object

Options for scraping search results

scrapeOptions.formats

enum<string>[]

Formats to include in the output

Available options:

`markdown`,

`html`,

`rawHtml`,

`links`,

`screenshot`,

`screenshot@fullPage`,

`extract`

### Response

200

application/json

Successful response

success

boolean

data

object[]

data.title

string

Title from search result

data.description

string

Description from search result

data.url

string

URL of the search result

data.markdown

string | null

Markdown content if scraping was requested

data.html

string | null

HTML content if requested in formats

data.rawHtml

string | null

Raw HTML content if requested in formats

data.links

string[]

Links found if requested in formats

data.screenshot

string | null

Screenshot URL if requested in formats

data.metadata

object

data.metadata.title

string

data.metadata.description

string

data.metadata.sourceURL

string

data.metadata.statusCode

integer

data.metadata.error

string | null

warning

string | null

# Perplexity Sonar API Documentation

POST

/chat/completions

```
curl --request POST \
  --url https://api.perplexity.ai/chat/completions \
  --header 'Authorization: Bearer <token>' \
  --header 'Content-Type: application/json' \
  --data '{
  "model": "sonar",
  "messages": [\
    {\
      "role": "system",\
      "content": "Be precise and concise."\
    },\
    {\
      "role": "user",\
      "content": "How many stars are there in our galaxy?"\
    }\
  ],
  "max_tokens": 123,
  "temperature": 0.2,
  "top_p": 0.9,
  "search_domain_filter": null,
  "return_images": false,
  "return_related_questions": false,
  "search_recency_filter": "<string>",
  "top_k": 0,
  "stream": false,
  "presence_penalty": 0,
  "frequency_penalty": 1,
  "response_format": null
}'
```

```
{
  "id": "3c90c3cc-0d44-4b50-8888-8dd25736052a",
  "model": "sonar",
  "object": "chat.completion",
  "created": 1724369245,
  "citations": [\
    "https://www.astronomy.com/science/astro-for-kids-how-many-stars-are-there-in-space/",\
    "https://www.esa.int/Science_Exploration/Space_Science/Herschel/How_many_stars_are_there_in_the_Universe",\
    "https://www.space.com/25959-how-many-stars-are-in-the-milky-way.html",\
    "https://www.space.com/26078-how-many-stars-are-there.html",\
    "https://en.wikipedia.org/wiki/Milky_Way"\
  ],
  "choices": [\
    {\
      "index": 0,\
      "finish_reason": "stop",\
      "message": {\
        "role": "assistant",\
        "content": "The number of stars in the Milky Way galaxy is estimated to be between 100 billion and 400 billion stars. The most recent estimates from the Gaia mission suggest that there are approximately 100 to 400 billion stars in the Milky Way, with significant uncertainties remaining due to the difficulty in detecting faint red dwarfs and brown dwarfs."\
      },\
      "delta": {\
        "role": "assistant",\
        "content": ""\
      }\
    }\
  ],
  "usage": {
    "prompt_tokens": 14,
    "completion_tokens": 70,
    "total_tokens": 84
  }
}
```

### Authorizations

Authorization

string

header
