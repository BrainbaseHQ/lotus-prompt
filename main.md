How to Write Based Agents
Overview
Based is a Python-like DSL for building platform-agnostic conversational AI agents with tool calling capabilities. Agents use a main conversation loop with talk() and trigger tools via until clauses.
Core Language Constructs
1. The say() Function
Purpose: Generate AI responses to users without expecting a reply
Syntax: say(message, exact=False, model=None)
Parameters:
message (string): Content to be processed and presented
exact (boolean, optional):
True: Output exactly as provided (verbatim)
False (default): Allow AI to rephrase while maintaining meaning
model (string, optional): Specify which AI model to use
Usage:
say("Welcome to the service!", exact=True)
say("Generate a friendly greeting for the user")
response = say("Introduce yourself as a helpful assistant")
2. The loop-talk-until Pattern
Critical Rule: talk MUST ALWAYS be used within a loop-until structure, never in isolation
Purpose: Create interactive conversation flows that repeat until conditions are met
Syntax:
loop:
   response = talk(system_prompt, first_prompt=True, default_values={}, info={})
until "Natural language description of completion condition":
   # Validation code that determines if condition is met
   # Loop continues until this code block completes successfully

Parameters for talk:
system_prompt (string): Instruction/prompt guiding the conversation
first_prompt (boolean):
True (default): AI starts by sending the prompt
False: AI waits for user to send message first
default_values (dict): Example values for structuring responses
info (dict): Additional context for the conversation
3. Data Extraction with .ask() Method
Purpose: Extract structured data from any data object (responses, API results, etc.)
Syntax: object.ask(question="...", example={...})
Parameters:
question (string): What information to extract
example (dict): Structure/format of expected output
Usage:
user_info = response.ask(
   question="Extract the user's name and email",
   example={"name": "John Doe", "email": "john@example.com"}
)
4. Content Summarization with .summarize() Method
Purpose: Create concise summaries of content
Syntax: object.summarize(focus="...", format="...")
Usage:
summary = long_content.summarize(
   focus="key points about user preferences",
   format="bullet_points"
)
Core Structure
import datetime
import uuid

# 1. STATE - Store conversation data
state['session_id'] = str(uuid.uuid4())
state['selected_item'] = None
state['customer_info'] = {}

# 2. CONFIG - API credentials, constants
BASE_URL = "https://api.example.com"
API_KEY = "your_api_key"

# 3. DATA - Hardcode fixed information (no API needed)
PRODUCTS = {
"product_a": {"id": 123, "name": "Product A", "price": 100},
"product_b": {"id": 456, "name": "Product B", "price": 200}
}

# 4. MAIN PROMPT - Everything the agent needs to know
MAIN_PROMPT = """You are a helpful assistant for [Company].

PRODUCTS:
• Product A - $100
• Product B - $200

To [ACTION]: Need [required info]
To [ACTION]: Need [required info]

Collect all required info before calling tools. Be conversational."""

# 5. GREETING
say("Hello! Welcome to [Company]. How can I help you today?")

# 6. MAIN LOOP
loop:
res = talk(MAIN_PROMPT + f" Today is {datetime.datetime.now().date()}.", False)

# 7. TOOLS (until clauses)
until "trigger description":
data = res.ask(question="What to extract?", example={"field": "value"})
# Process and call API
say("Response to customer")
continue
Key Concepts
1. State Management
state['key'] = value # Persists across conversation turns

Use the global state object to maintain conversation state
Access with state.get('key', default_value)
Store data with state['key'] = value
2. The Main Loop
loop:
res = talk(PROMPT, False) # False = don't require response

One main talk() that contains all context
Agent answers questions naturally from the prompt
Only calls tools when specific intents are detected
3. Tool Triggers (until clauses)
until "customer wants to [action] and has provided [required info]":
data = res.ask(question="...", example={...})
# Tool logic
continue # Return to main loop
Key principle: The until trigger should specify WHAT info is needed
❌ "customer wants to check availability" - might trigger without date
✅ "customer wants to check availability with tour and date provided" - only triggers when ready
4. Nested Loops/Untils
Nested loops/untils are primarily useful when you need a more deterministic flow to an agent. It is not required when building simple agents with relatively straightforward objectives.
When to use nested loops/untils:
You have a very specific, complex flow to follow
You have actions the LLM needs to take that rely on previous actions
You need to collect specific information before making an API call
You need sequential validation steps
# Example: Nested flow for multi-step verification
loop:
res = talk(MAIN_PROMPT, False)
until "customer wants to verify identity":
   loop:
       verification_res = talk("Please provide your account number", True)
   until "account number provided":
       account = verification_res.ask(question="Account number?", example={"account": "12345"})
       # Verify account exists via API
       loop:
           dob_res = talk("Please provide your date of birth for verification", True)
       until "date of birth provided":
           dob = dob_res.ask(question="DOB?", example={"dob": "1990-01-15"})
           # Now verify both pieces match
           break
       break
   continue
5. Extracting Data with res.ask()
data = res.ask(
question="What info to extract from the conversation?",
example={"field1": "example_value", "field2": 123}
)
# Access: data.get('field1'), data.get('field2')
6. API Calls
# GET request
response = api.get_req(url=url, headers=headers)
result = response.get('data', response) # Handle wrapper

# POST request
response = api.post_req(url=url, headers=headers, data=body)
result = response.get('data', response)
7. Speaking to Customer
say("Message to customer") # Simple message
say(f"Dynamic message with {variable}") # With variables
say("Exact message", exact=True) # Verbatim output
8. Flow Control
continue # Return to main loop after tool completes
break # Exit current loop
# No break needed in main loop - conversation continues until ended
Voice Platform Features
Email Confirmation TTS Pronunciation
When repeating back email addresses for confirmation, include this guidance in your prompt to ensure Text-to-Speech pronounces emails correctly:
ONLY when repeating back an email for CONFIRMATION (not when saying an email in conversation): Repeat it back character by character in all caps with a comma in between them, until the domain name. dot and at shouldn't be capitalized. Symbols should be pronounced as their full name. '.' should be 'dot', and '@' should be 'at'.
Example: matthew.cohen@gmail.com becomes: M, A, T, T, H, E, W, dot, C, O, H, E, N, at, gmail.com
Rules:
Never include markdown or special characters such as : or *
If the domain name has words inside, use the words (e.g., gmail.com)
If the domain name is unique (and has no words inside), repeat it back character by character with periods and commas in between
Platform-Specific Functions
transfer("+1234567890")  # Transfer call to phone number
time.sleep(seconds)      # Add delays in conversation flow (use before transfers or API calls)
Design Principles
1. AI-First: Don't API What You Can Hardcode
If data is fixed (product list, prices, descriptions), put it in the prompt. Only use APIs for dynamic data (availability, bookings, real-time info).
# ❌ Don't do this - unnecessary API call
until "customer asks about products":
products = api.get_req(url="/products")
say(f"We have: {products}")

# ✅ Do this - info is in the prompt
MAIN_PROMPT = """
PRODUCTS:
• Widget A - $50
• Widget B - $75
"""
# Agent answers naturally, no tool needed
2. Gather Before Calling
The agent should collect ALL required info before triggering a tool.
# ❌ Bad - triggers without complete info
until "customer wants to book":
data = res.ask(...)
if not data.get('date'):
say("What date?") # Now stuck in wrong flow
continue

# ✅ Good - trigger specifies requirements
until "customer provides all booking details including date, guests, and contact info":
data = res.ask(...)
# All info guaranteed to be present
3. Conversational Prompt
Write prompts that guide natural conversation:
MAIN_PROMPT = """You are a friendly assistant for Acme Tours.

WHAT YOU CAN DO:
• Show available tours and prices
• Check availability for specific dates
• Make bookings
• Look up existing bookings

BOOKING REQUIREMENTS:
To book, you need: tour name, date, number of guests, name, email, phone, pickup location

Be warm and helpful. Ask clarifying questions naturally."""
4. Handle Missing Info Gracefully
until "customer wants to book with all details":
data = res.ask(question="Get all booking fields", example={...})

missing = [f for f in ['name', 'email', 'phone'] if not data.get(f)]
if missing:
say(f"I still need your {', '.join(missing)}.")
continue # Back to main loop to collect more

# Proceed with booking
API Authentication Patterns
Simple Header Auth
headers = {'Authorization': f'Bearer {API_KEY}', 'Content-Type': 'application/json'}
HMAC Signature Auth (like Bokun)
import hmac
import hashlib
import base64

date_header = datetime.datetime.utcnow().strftime('%Y-%m-%d %H:%M:%S')
string_to_sign = f"{date_header}{ACCESS_KEY}{METHOD}{path}"
signature = hmac.new(SECRET_KEY.encode(), string_to_sign.encode(), hashlib.sha1).digest()
signature_b64 = base64.b64encode(signature).decode()

headers = {'X-Api-Key': ACCESS_KEY, 'X-Api-Date': date_header, 'X-Api-Signature': signature_b64}
Formatting Rules
No blank lines between statements (parser limitation)
Single-line statements - no multi-line function calls
Inline dicts/lists when possible
# ❌ Don't do this
headers = {
'Content-Type': 'application/json',
'Authorization': 'Bearer xxx'
}

# ✅ Do this
headers = {'Content-Type': 'application/json', 'Authorization': 'Bearer xxx'}
Best Practices
1. Conversation Flow Design
Start each major interaction with a clear say() statement
Use the loop-talk-until pattern for all interactive exchanges
Provide clear, natural language descriptions in until clauses
Always validate user input before proceeding
2. Data Handling
Use .ask() method to extract structured data from all responses
Provide comprehensive example objects to ensure proper data structure
Handle missing or invalid data gracefully
Store important data in the state object for persistence
3. Error Handling
Implement retry limits to prevent infinite loops
Provide fallback options (human transfer, alternative flows)
Use time.sleep() before transfers or API calls
Handle API failures gracefully
4. Code Organization
Define constants at the top of the file (URLs, credentials, prompts)
Use descriptive variable names
Group related functionality together
Add comments for complex business logic
5. Security Considerations
Never expose sensitive data in prompts or responses
Validate user input thoroughly
Use secure API headers and authentication
Implement proper data masking for PII
Common Anti-Patterns to Avoid
Using talk() without loop-until: Always wrap talk() in the proper structure
Missing data validation: Always validate extracted data before using it
Infinite loops: Implement retry limits and exit conditions
Exposing sensitive data: Never include actual user data in system prompts
Ignoring API errors: Always handle potential API failures
Missing state management: Use state object for conversation persistence
File Structure Guidelines
# 1. Imports and constants
import datetime
import re

# 2. Configuration variables
USERNAME = "service_user"
PASSWORD = "secure_password"
API_BASE_URL = "https://api.example.com"

# 3. Helper functions and prompts
validation_prompt = """
Your validation instructions here...
"""

# 4. Main conversation flow
say("Initial greeting")

# 5. Account lookup and validation
# ... implementation ...

# 6. Main interaction loops
loop:
   response = talk("Main interaction prompt", True)
until "Interaction complete":
   # Process response
   pass

# 7. Cleanup and transfer/completion
say("Closing message")
transfer("+1234567890")
Complete Example Template
import datetime
import uuid
import hmac
import hashlib
import base64
state['session_id'] = str(uuid.uuid4())
state['selected_item'] = None
state['customer_info'] = {}
BASE_URL = "https://api.example.com"
API_KEY = "your_key"
ITEMS = {"item_a": {"id": 1, "name": "Item A", "price": 100}, "item_b": {"id": 2, "name": "Item B", "price": 200}}
MAIN_PROMPT = """You are a helpful assistant for Example Company.
PRODUCTS: Item A ($100), Item B ($200)
To CHECK AVAILABILITY: Need item name + date
To BOOK: Need item, date, name, email, phone
To LOOKUP: Need confirmation code
Be friendly and collect all info before calling tools."""
say("Hi! Welcome to Example Company. What can I help you with?")
loop:
res = talk(MAIN_PROMPT + f" Today is {datetime.datetime.now().date()}.", False)
until "customer wants to check availability with item and date":
data = res.ask(question="Which item and date?", example={"item": "item a", "date": "2025-12-15"})
item_name = data.get('item', '').lower()
item = None
for key, i in ITEMS.items():
if item_name in i['name'].lower():
item = i
state['selected_item'] = i
break
if not item:
say("Which item are you interested in?")
continue
if not data.get('date'):
say("What date?")
continue
say(f"Checking {data['date']}...")
headers = {'Authorization': f'Bearer {API_KEY}', 'Content-Type': 'application/json'}
response = api.get_req(url=f"{BASE_URL}/availability?item={item['id']}&date={data['date']}", headers=headers)
result = response.get('data', {})
if result.get('available'):
say(f"Available on {data['date']}! Want to book?")
else:
say(f"Not available. Try another date?")
continue
until "customer provides all booking details":
data = res.ask(question="Get: name, email, phone", example={"name": "John", "email": "j@test.com", "phone": "+1234"})
if not state.get('selected_item'):
say("Let's first pick an item.")
continue
missing = [f for f in ['name', 'email', 'phone'] if not data.get(f)]
if missing:
say(f"I need your {', '.join(missing)}.")
continue
say("Creating booking...")
headers = {'Authorization': f'Bearer {API_KEY}', 'Content-Type': 'application/json'}
body = {"itemId": state['selected_item']['id'], "customer": {"name": data['name'], "email": data['email'], "phone": data['phone']}}
response = api.post_req(url=f"{BASE_URL}/bookings", headers=headers, data=body)
result = response.get('data', {})
if result.get('confirmationCode'):
say(f"Booked! Code: {result['confirmationCode']}")
else:
say("Something went wrong. Please try again.")
continue
until "customer wants to lookup booking with code":
data = res.ask(question="Confirmation code?", example={"code": "ABC123"})
if not data.get('code'):
say("I need your confirmation code.")
continue
say("Looking up...")
headers = {'Authorization': f'Bearer {API_KEY}'}
response = api.get_req(url=f"{BASE_URL}/bookings/{data['code']}", headers=headers)
result = response.get('data', {})
if result.get('id'):
say(f"Found: {result.get('itemName')} on {result.get('date')}.")
else:
say("Couldn't find that booking.")
continue
Testing and Debugging
Use print() statements for debugging (they won't appear to users)
Test with various input scenarios
Verify API integrations work correctly
Test error conditions and edge cases
Debugging Tips:
API not returning data? Check response structure:
response.get('data', response) handles wrapper
result.get('response', result) handles nested wrapper
Tool triggering incorrectly? Make trigger more specific:
Add "with [required info] provided" to trigger text
State not persisting? Use state['key'] not local variables
Formatting errors? Check for blank lines or multi-line statements
Platform Deployment Considerations
Based scripts are platform-agnostic
Test on target platforms (voice, SMS, chat) before deployment
Consider platform-specific limitations (character limits, etc.)
Use appropriate delays for voice platforms
Remember: Based is designed to be declarative - focus on "what" the conversation should accomplish rather than "how" to implement the technical details.

