# Creating a Meta Llama API Client in Python

Based on the Meta Llama developer documentation, I'll create a comprehensive Python client that demonstrates all possible API request values. This assumes you have access to Meta's Llama API through their developer portal.

## Prerequisites

1. Sign up for API access at [Meta Llama Developer Portal](https://llama.developer.meta.com)
2. Get your API key
3. Install required packages:
```bash
pip install requests python-dotenv
```

## Complete API Client Implementation

```python
import os
import requests
from dotenv import load_dotenv
from typing import Optional, Dict, Any, List, Union

# Load environment variables
load_dotenv()

class MetaLlamaClient:
    """
    A comprehensive client for Meta's Llama API demonstrating all possible request parameters.
    """
    
    def __init__(self, api_key: Optional[str] = None):
        """
        Initialize the client with API key.
        
        Args:
            api_key (str, optional): Your Meta Llama API key. If not provided,
                                   will try to get from META_LLAMA_API_KEY environment variable.
        """
        self.base_url = "https://api.llama.meta.com/v1"
        self.api_key = api_key or os.getenv("META_LLAMA_API_KEY")
        if not self.api_key:
            raise ValueError("API key must be provided or set in environment variables")
        
        self.headers = {
            "Authorization": f"Bearer {self.api_key}",
            "Content-Type": "application/json"
        }
    
    def generate_completion(
        self,
        model: str = "llama-3-8b",
        prompt: str = "",
        system_prompt: Optional[str] = None,
        messages: Optional[List[Dict[str, str]]] = None,
        max_tokens: int = 100,
        temperature: float = 0.7,
        top_p: float = 0.9,
        frequency_penalty: float = 0.0,
        presence_penalty: float = 0.0,
        stop_sequences: Optional[List[str]] = None,
        logprobs: Optional[int] = None,
        echo: bool = False,
        stream: bool = False,
        seed: Optional[int] = None,
        safe_prompt: bool = False,
        response_format: Optional[Dict[str, str]] = None,
        tools: Optional[List[Dict[str, Any]]] = None,
        tool_choice: Optional[Union[str, Dict[str, str]]] = None,
        images: Optional[List[str]] = None,
        **kwargs
    ) -> Dict[str, Any]:
        """
        Generate a completion using Meta's Llama API with all possible parameters.
        
        Args:
            model (str): The model ID to use (e.g., "llama-3-8b", "llama-4-scout")
            prompt (str): The input prompt text
            system_prompt (str, optional): System message to set model behavior
            messages (list, optional): List of message dicts for chat format
            max_tokens (int): Maximum number of tokens to generate
            temperature (float): Sampling temperature (0-2)
            top_p (float): Nucleus sampling threshold (0-1)
            frequency_penalty (float): Penalize frequent tokens (-2 to 2)
            presence_penalty (float): Penalize new tokens (-2 to 2)
            stop_sequences (list, optional): Sequences where generation stops
            logprobs (int, optional): Return top n log probabilities
            echo (bool): Echo back the prompt in the response
            stream (bool): Stream response chunks
            seed (int, optional): Random seed for reproducibility
            safe_prompt (bool): Apply safety filters
            response_format (dict, optional): Format like {"type": "json_object"}
            tools (list, optional): List of tools/functions available to model
            tool_choice (str/dict, optional): Which tool to use ("auto", "none", or specific tool)
            images (list, optional): Base64 encoded images for multimodal models
            **kwargs: Additional parameters
            
        Returns:
            dict: API response containing completion and metadata
        """
        endpoint = f"{self.base_url}/completions"
        
        payload = {
            "model": model,
            "prompt": prompt,
            "max_tokens": max_tokens,
            "temperature": temperature,
            "top_p": top_p,
            "frequency_penalty": frequency_penalty,
            "presence_penalty": presence_penalty,
            "echo": echo,
            "stream": stream,
            "safe_prompt": safe_prompt,
            **kwargs
        }
        
        # Optional parameters
        if system_prompt:
            payload["system_prompt"] = system_prompt
        if messages:
            payload["messages"] = messages
        if stop_sequences:
            payload["stop"] = stop_sequences
        if logprobs:
            payload["logprobs"] = logprobs
        if seed:
            payload["seed"] = seed
        if response_format:
            payload["response_format"] = response_format
        if tools:
            payload["tools"] = tools
        if tool_choice:
            payload["tool_choice"] = tool_choice
        if images:
            payload["images"] = images
        
        response = requests.post(
            endpoint,
            headers=self.headers,
            json=payload,
            stream=stream
        )
        
        if stream:
            return self._handle_streaming_response(response)
        else:
            return self._handle_response(response)
    
    def _handle_response(self, response: requests.Response) -> Dict[str, Any]:
        """Handle non-streaming responses."""
        if response.status_code != 200:
            raise Exception(f"API request failed with status {response.status_code}: {response.text}")
        return response.json()
    
    def _handle_streaming_response(self, response: requests.Response):
        """Handle streaming responses."""
        if response.status_code != 200:
            raise Exception(f"API request failed with status {response.status_code}: {response.text}")
        
        for chunk in response.iter_lines():
            if chunk:
                decoded_chunk = chunk.decode('utf-8')
                if decoded_chunk.startswith('data: '):
                    yield decoded_chunk[6:]  # Remove 'data: ' prefix

    def list_models(self) -> Dict[str, Any]:
        """List available models."""
        endpoint = f"{self.base_url}/models"
        response = requests.get(endpoint, headers=self.headers)
        return self._handle_response(response)
    
    def get_model_info(self, model_id: str) -> Dict[str, Any]:
        """Get information about a specific model."""
        endpoint = f"{self.base_url}/models/{model_id}"
        response = requests.get(endpoint, headers=self.headers)
        return self._handle_response(response)


# Example Usage
if __name__ == "__main__":
    # Initialize client (API key from environment variable)
    client = MetaLlamaClient()
    
    # Example 1: Basic completion
    print("=== BASIC COMPLETION ===")
    basic_response = client.generate_completion(
        prompt="Explain quantum computing in simple terms",
        max_tokens=150,
        temperature=0.8
    )
    print(basic_response["choices"][0]["text"])
    
    # Example 2: Chat completion with messages
    print("\n=== CHAT COMPLETION ===")
    chat_response = client.generate_completion(
        model="llama-4-scout",
        messages=[
            {"role": "system", "content": "You are a helpful assistant."},
            {"role": "user", "content": "What's the weather like today?"}
        ],
        max_tokens=100
    )
    print(chat_response["choices"][0]["message"]["content"])
    
    # Example 3: JSON response format
    print("\n=== JSON RESPONSE ===")
    json_response = client.generate_completion(
        prompt="List 3 fruits and their colors in JSON format",
        response_format={"type": "json_object"},
        max_tokens=200
    )
    print(json_response["choices"][0]["text"])
    
    # Example 4: Streaming response
    print("\n=== STREAMING RESPONSE ===")
    for chunk in client.generate_completion(
        prompt="Tell me a short story about a dragon",
        stream=True,
        max_tokens=300
    ):
        print(chunk, end="", flush=True)
    
    # Get available models
    print("\n\n=== AVAILABLE MODELS ===")
    models = client.list_models()
    print("Available models:", [m["id"] for m in models["data"]])
```

## Key Features Demonstrated

1. **All API Parameters**:
   - Model selection (`llama-3-8b`, `llama-4-scout`, etc.)
   - Prompt engineering (system prompts, message history)
   - Generation controls (temperature, top_p, max_tokens)
   - Safety controls (safe_prompt)
   - Advanced features (logprobs, seed for reproducibility)
   - Multimodal support (images parameter)
   - Function calling (tools and tool_choice)

2. **Response Handling**:
   - Standard JSON responses
   - Streaming responses for real-time output
   - Error handling

3. **Additional API Features**:
   - Model listing
   - Model information retrieval

## Usage Instructions

1. Save this code to a file (e.g., `meta_llama_client.py`)
2. Create a `.env` file with your API key:
   ```
   META_LLAMA_API_KEY=your_api_key_here
   ```
3. Run the script to test different API features

Note: The exact parameter names and available models may vary slightly based on the specific version of the Meta Llama API you're accessing. Always refer to the latest [official documentation](https://llama.developer.meta.com/docs) for the most accurate information.