# Text-to-SQL Generative AI Chatbot

An intelligent, self-correcting Text-to-SQL system that converts natural language questions into executable SQL queries. Built with LangGraph and Google's Generative AI, this system can understand database schemas, generate SQL, execute queries, and automatically fix errors through iterative feedback.

## Problem Statement

The promise of Text-to-SQL systems is straightforward: ask your database a question in plain English and get an answer back. While creating a demo is easy, production databases present real challenges:

- **Schema Complexity**: Poorly designed schemas, tables with unclear naming conventions, and redundant table relationships
- **Column Issues**: Column names with spaces, inconsistent naming patterns, incomplete documentation
- **LLM Limitations**: Large Language Models can generate syntactically incorrect SQL that fails at execution
- **Traditional Approach Limitation**: Standard Text-to-SQL systems treat SQL generation as a one-off translation task—if it fails, the user gets a database error

## Solution: Self-Correcting Agentic System

Rather than viewing Text-to-SQL as a simple translation problem, we approach it as a **navigation and feedback task**. The system:

1. **Generates SQL** based on the user's question and database schema
2. **Executes the query** immediately
3. **Captures any errors** that occur during execution
4. **Self-corrects** by providing the error message back to the LLM as feedback
5. **Regenerates improved SQL** up to 3 iterations until success

This transforms a fragile proof-of-concept into a **resilient, production-ready system**.

## Architecture

### System Components

```
User Question
    ↓
[Schema Retrieval] → Fetch relevant database schema
    ↓
[SQL Generation] → LLM generates SQL query
    ↓
[Query Execution] → Execute SQL on database
    ↓
[Error Router] → Check execution result
    ├─ Success → Return result (END)
    └─ Error → Send error feedback → [SQL Generation] (Re-attempt)
    ↓
[Final Output] → Return result or final error after max retries
```

### Key Components Explained

1. **AgentState**: Tracks the conversation state including question, schema, SQL query, results, errors, and iteration count
2. **Schema Node**: Retrieves database table information and column definitions
3. **SQL Generation Node**: Uses Google's Gemini AI to generate contextual SQL queries
4. **Execution Node**: Runs the SQL on the MySQL database and captures errors
5. **Router**: Decides whether to retry with error feedback or return final result

## Prerequisites

Before running this project, ensure you have:

- **Python 3.8+** installed
- **MySQL Server** running locally with a database and tables
- **Google API Key** for Generative AI access
- **pip** package manager

### Required Libraries

- `langchain` - LLM orchestration framework
- `langchain-google-genai` - Google Generative AI integration
- `langchain-community` - Community LangChain utilities
- `langgraph` - Agentic workflow framework
- `python-dotenv` - Environment variable management
- `pymysql` - MySQL database driver

## Setup Instructions

### 1. Clone/Download the Repository

```bash
cd c:\Northeastern\Project\Text2SQL
```

### 2. Install Dependencies

```bash
pip install langchain langchain-google-genai langchain-community langgraph python-dotenv pymysql
```

### 3. Configure Google API Key

Create a `.env` file in the project root:

```
GOOGLE_API_KEY=your_google_api_key_here
```

**How to get your Google API Key**:
1. Go to [Google AI Studio](https://aistudio.google.com/apikey)
2. Click "Create API Key"
3. Copy the key and paste it in the `.env` file

### 4. Configure Database Connection

In `text2sql.ipynb`, update the MySQL connection string:

```python
mysql_uri = "mysql+pymysql://your_username:your_password@localhost:3306/your_database_name"
```

Replace with your actual:
- `your_username` - MySQL username
- `your_password` - MySQL password
- `your_database_name` - Target database name

### 5. Verify Database Setup

Ensure your MySQL database has tables with proper schemas. The system will automatically fetch schema information.

## How to Use

### Running the Notebook

1. Open `text2sql.ipynb` in Jupyter Notebook or VS Code
2. Run all cells sequentially (Shift+Enter for each cell)
3. The system will execute two test queries by default

### Example Usage

```python
# Ask a natural language question
test_question = "What was the total sale for the East region?"

# Invoke the workflow
final_state = app.invoke({"question": test_question})

# Access results
print(f"Question: {test_question}")
print(f"Generated SQL: {final_state['sql_query']}")
print(f"Result: {final_state['db_result']}")
print(f"Iterations: {final_state['iteration_count']}")
```

### Understanding the Output

- **SQL GENERATED**: The final SQL query created (possibly after corrections)
- **RESULT**: The database query result (rows returned)
- **ITERATIONS**: Number of attempts needed (1 = first try success, 2+ = error correction occurred)
- **FINAL ERROR**: If present, indicates query still failed after max retries

## End-to-End Workflow Example

### Scenario: Asking "What was the total sale for Category 3 in East region?"

**Step 1: Schema Retrieval**
```
Tables: sales, categories, regions
Columns: sale_id, amount, category_id, region_id, date, ...
```

**Step 2: Initial SQL Generation**
```sql
SELECT SUM(amount) FROM sales 
WHERE category_id = 3 AND region_id = 'East'
```

**Step 3: Execution Attempt 1**
- If successful → Return result
- If error → Capture error message (e.g., "Unknown column 'region_id'")

**Step 4: Error Feedback Loop**
```
ERROR: "Unknown column 'region_id'"
FEEDBACK: "The column might be 'region' or related through a join"
```

**Step 5: Regenerated SQL (Iteration 2)**
```sql
SELECT SUM(s.amount) FROM sales s
JOIN regions r ON s.region_id = r.id
WHERE s.category_id = 3 AND r.name = 'East'
```

**Step 6: Final Execution**
- Success → Return result
- Or continue retry (max 3 iterations)

## Code Structure Breakdown

### Cell 1: Imports
Imports all necessary libraries for LLM integration, database connection, and workflow orchestration.

### Cell 2: Database & LLM Configuration
Establishes MySQL connection and initializes Google's Gemma-3 model as the LLM.

### Cell 3: AgentState Definition
Defines the data structure that flows through the workflow with fields for tracking question, schema, SQL, results, and errors.

### Cell 4: Schema Retrieval Node
Fetches table and column information from the database. Could be optimized with vector similarity for large schemas.

### Cell 5: SQL Generation Node
- Constructs a prompt with schema and question
- Includes previous error context if a retry is happening
- Parses the model's response to extract SQL from markdown code blocks

### Cell 6: Execution Node
Runs the SQL safely and captures exceptions as error messages for feedback.

### Cell 7: Router Logic
Decides workflow direction:
- Stop if query executed successfully
- Retry with feedback if errors occur and iterations < 3
- Stop after 3 failed attempts to prevent infinite loops

### Cell 8: Workflow Compilation
Builds the LangGraph workflow by connecting all nodes with conditional routing logic.

### Cells 9-10: Test Cases
Executes sample queries to demonstrate the system's capabilities.


### Adjusting Max Iterations

Modify the `router()` function to change retry attempts:

```python
if state["iteration_count"] >= 5:  # Increased from 3 to 5
    return "end"
```

### Changing LLM Model

Update the LLM initialization in Cell 2:

```python
llm = ChatGoogleGenerativeAI(model="gemini-2.0-flash", temperature=0)
```

### Schema Optimization

For large databases, implement schema pruning to fetch only relevant tables:

```python
# Instead of db.get_table_info(), use vector similarity search
relevant_tables = vector_db.similarity_search(question, k=5)
```

## Common Issues & Troubleshooting

### Issue: "Invalid API Key"
**Solution**: Verify your Google API key in `.env` file and ensure it has Generative AI permissions enabled.

### Issue: "Cannot connect to MySQL"
**Solution**: Check that:
- MySQL server is running
- Username and password are correct
- Database name exists
- No firewall blocking localhost:3306

### Issue: "Unknown column" Errors Keep Occurring
**Solution**: 
- Verify column names in your database schema
- Check for typos in table/column names
- The system will attempt corrections, but extreme schema inconsistencies may exceed 3 iterations

### Issue: Query Returns Empty Results
**Solution**: 
- The query may be syntactically correct but logically wrong
- Review the generated SQL in output
- Verify data exists for the queried conditions
- Consider asking more specific questions

## Performance Considerations

- **Single Query Latency**: ~2-5 seconds for successful first attempts, 5-15 seconds for error corrections
- **Database Impact**: Each iteration runs a new query, so retries increase database load
- **LLM Calls**: Each iteration calls the LLM, incurring API costs

## Learning Outcomes

This project demonstrates:
- Building agentic AI systems with LangGraph
- Iterative error correction patterns
- LLM prompt engineering with context injection
- Database integration with Python
- State management in complex workflows

## Future Enhancements

1. **Schema Pruning**: Use vector embeddings to fetch only relevant tables
2. **Multi-turn Context**: Remember previous queries in conversation
3. **Query Result Presentation**: Format results as natural language summaries
4. **Performance Optimization**: Cache frequently used schemas
5. **Extended Error Analysis**: Support more complex error patterns
6. **SQL Validation**: Pre-validation of syntax before execution
7. **Audit Logging**: Track all queries and corrections for compliance

## License

This project is registered under MIT license.

