# Deal Hunter
 
A personal deal-monitoring tool that watches hardware marketplaces, deal sites, and community forums for items on a personal wishlist. When a listing matches target specs and price thresholds, it sends a push notification.
 
Built for personal use on a local network, not a hosted service and not a commercial product.
 
## What It Does
 
- Monitors sources like eBay, Amazon, Newegg, Slickdeals, and Reddit communities (r/buildapcsales, r/hardwareswap, r/appleswap) for hardware deals
- Matches listings against a configurable wishlist with target prices, condition requirements, and spec constraints
- Scores candidates using a local LLM (Ollama) to evaluate spec accuracy, condition, and value
- Tracks price history over time to contextualize whether a deal is actually good
- Sends push notifications via ntfy.sh when a deal meets alert thresholds
- Provides a lightweight dashboard for reviewing deals, price trends, and alert history
 
## How It Uses Reddit
 
The Reddit integration is **read-only**. It polls a small number of subreddits for new posts, checks titles and bodies against the wishlist using keyword matching, and flags potential deals for further scoring. It does not post, comment, vote, or interact with other users in any way.
 
## Architecture
 
- **Python 3.13** with FastAPI, SQLite, and async HTTP via httpx
- **Ollama** (Qwen 2.5 14B) for local LLM-based deal scoring
- **ntfy.sh** for push notifications
- Runs on a local network behind Tailscale — not publicly accessible
 
## Status
 
Under active development. This is a personal project and a learning exercise in building data pipelines, web scraping, and API integrations with Python.
 
## License
 
This project is for personal use.