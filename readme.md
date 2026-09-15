# Triply

## AI travel planning, from one sentence to a trip you can actually use

Triply is a full-stack travel planning assistant. Describe a trip in natural language and it combines live flight-status data, hotel research, an AI-generated day-by-day itinerary, budget guidance, and practical recommendations in one response.

![Triply travel planner interface](docs/triply-ui.png)

The interface is intentionally simple: enter a request such as:

> Plan a complete 7 days Japan trip from Delhi including flights, hotels, sightseeing, and a budget under 2 lakhs.

Triply turns that request into a structured travel plan. You can also start with one of the built-in prompts for Japan, Dubai, Thailand, or global flight data.

## What it does

- **Understands natural-language requests**: destination, origin, duration, budget, hotels, and sightseeing preferences can be written conversationally.
- **Finds live flight information**: AviationStack results include airline, flight number, status, airports, IATA codes, terminals, gates, schedules, and delays.
- **Resolves locations automatically**: city names, country names, airport names, and IATA codes are converted into airport routes. If only a destination is supplied, `DEFAULT_ORIGIN_IATA` is used.
- **Supports route and global searches**: ask for flights from or to a location, a specific route, or all/global flights.
- **Researches hotels**: Tavily searches the web for relevant hotel information and returns the top five sources with links and short excerpts.
- **Builds a practical itinerary**: Groq generates a complete, budget-aware itinerary from the request and the collected flight and hotel research.
- **Produces a polished final answer**: the final AI response is organized into trip summary, flight information, hotel suggestions, day-by-day itinerary, estimated budget, and final recommendations.
- **Keeps conversation threads**: PostgreSQL-backed LangGraph checkpoints preserve a thread ID, and the browser stores the current thread locally.
- **Works well with the result**: answers are rendered as Markdown and can be copied to the clipboard or downloaded as an A4 PDF.
- **Handles loading and failure states**: the UI shows progress while planning and surfaces validation, API, and server errors in the page.
- **Responsive browser UI**: the planner adapts to smaller screens and includes print-friendly PDF styling.

## How the agents work

Every request moves through a deterministic LangGraph pipeline:

```text
User request
	|
	v
Flight agent       -> AviationStack live flight/status data
	|
	v
Hotel agent        -> Tavily hotel research
	|
	v
Itinerary agent    -> Groq practical day-by-day plan
	|
	v
Final agent        -> structured travel answer
	|
	v
FastAPI JSON response -> Markdown result in the browser
```

The graph state carries the original query, flight results, hotel results, itinerary, messages, and an `llm_calls` counter. PostgreSQL is used by `PostgresSaver` as the LangGraph checkpointer.

## Stack

- **Frontend**: HTML, CSS, vanilla JavaScript, Marked, html2pdf.js
- **Web API**: FastAPI, Uvicorn, Jinja2
- **Orchestration**: LangGraph and LangChain
- **Language model**: Groq `openai/gpt-oss-120b`
- **Flight data**: AviationStack
- **Web research**: Tavily
- **Persistence**: PostgreSQL with `langgraph-checkpoint-postgres` and Psycopg
- **Runtime**: Python 3.12; Docker support included

## Requirements

- Python 3.12 or newer
- A PostgreSQL database reachable from the application
- API keys for Groq, Tavily, and AviationStack
- Network access to those services at runtime

## Setup

1. Create and activate a virtual environment:

   ```powershell
   python -m venv .venv
   .\.venv\Scripts\Activate.ps1
   ```

2. Install dependencies:

   ```powershell
   pip install -r requirements.txt
   ```

3. Create a `.env` file in the project root:

   ```dotenv
   GROQ_API_KEY=your_groq_api_key
   TAVILY_API_KEY=your_tavily_api_key
   AVIATIONSTACK_API_KEY=your_aviationstack_api_key
   DATABASE_URL=postgresql://user:password@host:5432/database
   DEFAULT_ORIGIN_IATA=IND
   ```

   `DATABASE_URL` is required for LangGraph checkpointing. The application adds `sslmode=require` automatically when it is missing.

4. Start the application:

   ```powershell
   python app.py
   ```

5. Open [http://127.0.0.1:8000](http://127.0.0.1:8000).

The health endpoint is available at [http://127.0.0.1:8000/health](http://127.0.0.1:8000/health).

## Docker

Build and run the container with the same environment variables:

```powershell
docker build -t triply .
docker run --rm -p 8000:8000 --env-file .env triply
```

The included `Dockerfile` uses Python 3.12, installs the requirements, exposes port `8000`, and starts Uvicorn on `0.0.0.0`.

## Using the API

The browser calls `POST /api/travel` with a JSON body:

```json
{
  "message": "Plan a 5 days Dubai trip from Delhi with hotels and sightseeing",
  "thread_id": null
}
```

`thread_id` may be omitted or set to the ID returned by an earlier request. A successful response includes:

```json
{
  "success": true,
  "thread_id": "user_...",
  "answer": "...",
  "flight_results": "...",
  "hotel_results": "...",
  "itinerary": "...",
  "llm_calls": 4
}
```

Empty messages return `400`. Unexpected provider, database, or application failures return `500` with an error message.

## Useful prompt patterns

```text
Plan a 7 days Japan trip from Delhi under 2 lakhs.
Plan a 5 days Dubai trip from Gorakhpur with flights, hotels, and sightseeing.
Show flights from Mumbai to Bangkok.
Show flights to London.
Give me all country flight info.
Plan a budget Thailand trip with hotels and sightseeing.
```

Location parsing supports common country aliases, preferred city airports, airport names, and direct three-letter IATA codes. The default origin is `IND` unless `DEFAULT_ORIGIN_IATA` is changed.

## Important data notes

- AviationStack supplies live/status flight data, not guaranteed ticket prices. Triply tells the user when pricing is unavailable; a fare-specific provider such as Amadeus would be needed for booking prices.
- Hotel results are web research snippets from Tavily, not reservations or verified availability.
- AI budgets and recommendations are planning estimates. Confirm schedules, prices, visa rules, and availability with official providers before booking.
- API keys and database credentials belong in `.env`; never commit that file.

## Project structure

```text
.
├── app.py                 # FastAPI routes, validation, static files, templates
├── backend.py             # LangGraph state, agents, Groq, PostgreSQL checkpointer
├── tools/
│   ├── flight_tool.py     # Location parsing and AviationStack integration
│   └── tavily_tool.py     # Hotel web research integration
├── templates/index.html   # Triply page structure
├── static/script.js       # API calls, thread storage, Markdown, copy, PDF
├── static/style.css       # Responsive visual design and print styles
├── docs/triply-ui.png     # Screenshot captured from the running UI
├── requirements.txt       # Python dependencies
├── Dockerfile             # Container build and startup command
└── test.py                # Small flight-search smoke script
```

## Development notes

`test.py` is a manual smoke script for `search_flights`; it requires the flight API key and network access. The main application also requires all four environment values before `backend.py` can initialize, because the LangGraph and PostgreSQL objects are created during import.
