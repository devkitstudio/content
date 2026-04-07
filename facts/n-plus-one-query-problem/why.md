The "N+1" problem happens when your code executes **one** database query to retrieve a list of records, and then executes **N** additional queries (one per record) to retrieve related data.

### The Scenario
Imagine you have a blog. You want to fetch 100 Articles, and display the Author's name for each article.

**The Junior Approach (Lazy Loading in a loop):**
1. Fetch 100 articles: `SELECT * FROM articles LIMIT 100;` (1 query)
2. The code loops through the 100 articles.
3. Inside the loop, it fetches the author for the current article: `SELECT * FROM users WHERE id = X;` (This runs 100 times!)

**Total Queries Executed: 101.**
If you have 10,000 articles, that's 10,001 network trips to the database. The network latency alone will crush your server.
