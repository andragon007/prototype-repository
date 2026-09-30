import os
import subprocess
import tempfile
from dotenv import load_dotenv
from openai import OpenAI

# Load environment variables
load_dotenv()

# 1. Read the user's choices from the .env file
provider = os.getenv("AI_PROVIDER", "groq")
api_key = os.getenv("AI_API_KEY")
model_name = os.getenv("AI_MODEL", "openai/gpt-oss-120b")

# 2. Determine the "Address" (Base URL) based on the provider
if provider.lower() == "groq":
    base_url = "https://api.groq.com/openai/v1"
elif provider.lower() == "openai":
    base_url = "https://api.openai.com/v1"
elif provider.lower() == "local":
    base_url = "http://localhost:11434/v1" # Standard address for Ollama
else:
    base_url = "https://api.groq.com/openai/v1" # Default fallback

# 3. Create the client using the user's specific settings
client = OpenAI(api_key=api_key, base_url=base_url)

print(f"🚀 Connected to {provider} using model: {model_name}")

SYSTEM_PROMPT = """
You are an expert software engineer practicing "Design by Contract". 
When asked to write a function, you MUST follow these rules:

1. PRECONDITIONS: At the very beginning of the function, use standard Python `assert` statements to validate all inputs.
2. POSTCONDITIONS: At the very end of the function, use `assert` statements to validate the output.
3. TEST HARNESS: At the bottom of the script, write 3 distinct test cases including edge cases.

Do not include any explanations or markdown formatting. Output ONLY valid, executable Python code.
"""

def ask_ai_for_code(task_description, error_feedback=""):
    messages = [
        {"role": "system", "content": SYSTEM_PROMPT},
    ]
    
    if error_feedback:
        messages.append({"role": "user", "content": f"{task_description}\n\nYour previous code failed with this error:\n{error_feedback}\n\nFix the code and try again."})
    else:
        messages.append({"role": "user", "content": task_description})
    
    print(f" Asking AI to write code...\n")
    
    # Notice we now use the model_name variable from the .env file!
    response = client.chat.completions.create(
        model=model_name, 
        messages=messages,
        temperature=0.2
    )
    
    return response.choices[0].message.content

def execute_code_locally(code_string):
    with tempfile.NamedTemporaryFile(mode='w', suffix='.py', delete=False) as f:
        f.write(code_string)
        temp_file = f.name
    
    try:
        result = subprocess.run(["python", temp_file], capture_output=True, text=True, timeout=10)
        
        if result.returncode == 0:
            return True, ""
        else:
            return False, result.stderr.strip()
            
    except subprocess.TimeoutExpired:
        return False, "Execution timed out (possible infinite loop)"
    finally:
        os.remove(temp_file)

def generate_and_verify(task, max_retries=3):
    code = None
    last_error = ""
    
    for attempt in range(max_retries):
        print(f"\n{'='*50}")
        print(f"ATTEMPT {attempt + 1}/{max_retries}")
        print('='*50)
        
        code = ask_ai_for_code(task, last_error)
        
        print("\n--- Generated Code ---")
        print(code[:500] + "..." if len(code) > 500 else code)
        
        success, error = execute_code_locally(code)
        
        if success:
            print("\n✅ SUCCESS! Code passed all tests!")
            return code
        else:
            print(f"\n❌ FAILED: {error}")
            last_error = error
    
    print("\n⚠️ Max retries reached. Could not generate working code.")
    return code

if __name__ == "__main__":
    task = "Write a simple function called `add(a, b)` that returns the sum of two numbers. Include assert statements to verify both inputs are numbers. Add 2 simple test cases at the bottom."
    
    final_code = generate_and_verify(task)
