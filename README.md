"# Text-to-SQL-Generative-AI-Chatbot" 

The promise of Text-to-SQL systems is straightforward and very powerful: you could simply ask your database a question in plain English and get an answer back. Thanks to generative AI, it almost feels like magic, and creating a demo is unbelievably straightforward. However, the magic disappears as soon as you use real-world data.

Production databases, indeed, have quite a few issues. For example, schemas are not well thought out, column names sometimes have spaces, table names refer to the same concept as other tables, documentation is incomplete, and no one really knows where the business logic resides. Besides, LLMs are still prone to errors. Traditional Text-to-SQL approaches consider this issue a one-off translation problem: if the generated SQL is incorrect, the system abandons the user to a raw database error.

Moreover, the core idea is that Text-to-SQL is not only translation it is also navigation. Instead of interrogating whether an LLM can produce perfect SQL in one attempt, a more useful question might be whether the system is capable of recognizing its error and fixing itself.

Taking the problem as a search task that is driven by feedback, I made an agentic Text-to-SQL system with the help of LangGraph. First, the system writes SQL, runs it, and if an error occurs, the system can self-correct after the model is given the error message as feedback. It is a very clever way of turning a fragile demo into a resilient system.
