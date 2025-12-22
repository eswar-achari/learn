I'll create a comprehensive MCP tool for AIT test cases with MongoDB integration. Here's the complete implementation:

```python
import re
from typing import Optional, List, Dict, Any
from pymongo import MongoClient
from pydantic import BaseModel, Field
import os
from datetime import datetime

# Pydantic models for request/response validation
class AITTestCasesRequest(BaseModel):
    """Request model for fetching AIT test cases and suggestions."""
    ait: Optional[str] = Field(
        default=None,
        description="AIT identifier code (e.g., 'AIT-123')"
    )
    ait_name: Optional[str] = Field(
        default=None,
        description="Name of the AIT application"
    )
    category: Optional[str] = Field(
        default=None,
        description="Category for testing suggestions (e.g., 'web', 'mobile', 'api')"
    )

class TestingSuggestion(BaseModel):
    """Model for testing suggestion results."""
    category: str
    test_type: str
    suggestion: str
    priority: str
    complexity: str
    estimated_time: str
    tools_recommended: List[str]
    test_scenarios: List[str]

class AppSummary(BaseModel):
    """Model for application summary results."""
    ait: str
    ait_name: str
    category: str
    description: str
    tech_stack: List[str]
    dependencies: List[str]

class AITTestCasesResponse(BaseModel):
    """Response model for AIT test cases."""
    success: bool
    message: str
    app_summary: Optional[AppSummary] = None
    testing_suggestions: List[TestingSuggestion] = []
    matched_categories: List[str] = []
    timestamp: str

class MongoDBConfig:
    """MongoDB configuration manager."""
    def __init__(self):
        self.mongo_uri = os.getenv("MONGODB_URI", "mongodb://localhost:27017")
        self.database_name = os.getenv("MONGODB_DATABASE", "ait_testing_db")
        
    def get_client(self):
        """Create and return MongoDB client."""
        return MongoClient(self.mongo_uri)
    
    def get_database(self):
        """Get database instance."""
        client = self.get_client()
        return client[self.database_name]

class AITTestCasesRepository:
    """Repository for AIT test cases and suggestions."""
    
    def __init__(self):
        self.config = MongoDBConfig()
        self.db = self.config.get_database()
        
    def get_app_summary_by_ait(self, ait: str) -> Optional[Dict[str, Any]]:
        """Get application summary by AIT code."""
        collection = self.db['app_summary']
        return collection.find_one({"ait": {"$regex": f"^{ait}$", "$options": "i"}})
    
    def get_app_summary_by_name(self, ait_name: str) -> Optional[Dict[str, Any]]:
        """Get application summary by AIT name."""
        collection = self.db['app_summary']
        return collection.find_one({"ait_name": {"$regex": ait_name, "$options": "i"}})
    
    def get_testing_suggestions_by_category(self, category_pattern: str) -> List[Dict[str, Any]]:
        """Get testing suggestions by category regex pattern."""
        collection = self.db['testing_suggestions']
        # Create case-insensitive regex pattern
        regex_pattern = re.compile(category_pattern, re.IGNORECASE)
        cursor = collection.find({"category": {"$regex": regex_pattern}})
        return list(cursor)

class AITTestCasesTool:
    """MCP Tool for retrieving AIT test cases and testing suggestions."""
    
    name = "ait_test_cases"
    description = """
    Retrieves AIT (Application Integration Testing) test cases and testing suggestions 
    based on AIT code, application name, or category. 
    
    This tool helps QA teams and developers:
    1. Find relevant test cases for specific applications
    2. Get testing methodology suggestions based on application category
    3. Discover recommended tools and frameworks for testing
    
    Usage scenarios:
    - "Get test cases for AIT-123"
    - "Find testing suggestions for web applications"
    - "What test cases should I write for mobile app XYZ?"
    - "Get testing recommendations for API services"
    
    The tool intelligently routes queries:
    • If category is provided: Directly searches testing suggestions
    • If AIT or name is provided: First finds the app category, then gets suggestions
    """
    
    def __init__(self):
        self.repository = AITTestCasesRepository()
    
    def process_request(self, request: AITTestCasesRequest) -> AITTestCasesResponse:
        """Main processing method for AIT test cases requests."""
        
        app_summary = None
        testing_suggestions = []
        matched_categories = []
        
        try:
            # Case 1: Direct category search
            if request.category:
                suggestions = self.repository.get_testing_suggestions_by_category(request.category)
                testing_suggestions = suggestions
                matched_categories = list(set([s['category'] for s in suggestions]))
                
            # Case 2: Search by AIT or AIT name
            elif request.ait or request.ait_name:
                # Try to find app summary
                if request.ait:
                    app_summary = self.repository.get_app_summary_by_ait(request.ait)
                elif request.ait_name and not app_summary:
                    app_summary = self.repository.get_app_summary_by_name(request.ait_name)
                
                if app_summary:
                    # Get category from app summary and search suggestions
                    category = app_summary.get('category')
                    if category:
                        suggestions = self.repository.get_testing_suggestions_by_category(category)
                        testing_suggestions = suggestions
                        matched_categories = [category]
            
            # Prepare response
            response = AITTestCasesResponse(
                success=True,
                message=self._generate_message(
                    has_ait=bool(request.ait or request.ait_name),
                    has_category=bool(request.category),
                    found_app=bool(app_summary),
                    suggestion_count=len(testing_suggestions)
                ),
                app_summary=AppSummary(**app_summary) if app_summary else None,
                testing_suggestions=[TestingSuggestion(**s) for s in testing_suggestions],
                matched_categories=matched_categories,
                timestamp=datetime.now().isoformat()
            )
            
            return response
            
        except Exception as e:
            return AITTestCasesResponse(
                success=False,
                message=f"Error processing request: {str(e)}",
                app_summary=None,
                testing_suggestions=[],
                matched_categories=[],
                timestamp=datetime.now().isoformat()
            )
    
    def _generate_message(self, has_ait: bool, has_category: bool, 
                         found_app: bool, suggestion_count: int) -> str:
        """Generate appropriate response message based on search results."""
        
        if has_ait and not found_app:
            return "No application found with the specified AIT or name."
        
        if has_category and suggestion_count == 0:
            return "No testing suggestions found for the specified category."
        
        if suggestion_count > 0:
            return f"Found {suggestion_count} testing suggestion(s) based on your criteria."
        
        return "Please provide either AIT/name or category to get testing suggestions."

# MCP Server setup (for integration with MCP protocol)
class AITTestCasesMCPServer:
    """MCP Server wrapper for the AIT Test Cases tool."""
    
    def __init__(self):
        self.tool = AITTestCasesTool()
    
    def handle_request(self, params: Dict[str, Any]) -> Dict[str, Any]:
        """Handle MCP request and return formatted response."""
        
        # Parse request parameters
        request = AITTestCasesRequest(**params)
        
        # Process the request
        response = self.tool.process_request(request)
        
        # Format response for MCP
        return {
            "content": [
                {
                    "type": "text",
                    "text": self._format_response_for_display(response)
                }
            ],
            "isError": not response.success
        }
    
    def _format_response_for_display(self, response: AITTestCasesResponse) -> str:
        """Format the response for nice display in chat interfaces."""
        
        output = []
        
        if not response.success:
            return f"❌ {response.message}"
        
        output.append(f"## 📋 AIT Test Cases Results")
        output.append(f"**Status:** {response.message}")
        output.append(f"**Timestamp:** {response.timestamp}")
        
        if response.app_summary:
            output.append("\n## 🏢 Application Summary")
            app = response.app_summary
            output.append(f"**AIT:** {app.ait}")
            output.append(f"**Name:** {app.ait_name}")
            output.append(f"**Category:** {app.category}")
            output.append(f"**Description:** {app.description}")
            output.append(f"**Tech Stack:** {', '.join(app.tech_stack)}")
            output.append(f"**Dependencies:** {', '.join(app.dependencies)}")
        
        if response.testing_suggestions:
            output.append(f"\n## 🧪 Testing Suggestions ({len(response.testing_suggestions)} found)")
            
            for i, suggestion in enumerate(response.testing_suggestions, 1):
                output.append(f"\n### {i}. {suggestion.test_type} ({suggestion.category})")
                output.append(f"**Priority:** {suggestion.priority} | **Complexity:** {suggestion.complexity}")
                output.append(f"**Estimated Time:** {suggestion.estimated_time}")
                output.append(f"**Suggestion:** {suggestion.suggestion}")
                output.append(f"**Recommended Tools:** {', '.join(suggestion.tools_recommended)}")
                output.append("**Test Scenarios:**")
                for scenario in suggestion.test_scenarios:
                    output.append(f"  • {scenario}")
        
        if response.matched_categories:
            output.append(f"\n## 🏷️ Matched Categories")
            for category in response.matched_categories:
                output.append(f"• {category}")
        
        if not response.app_summary and not response.testing_suggestions:
            output.append("\n💡 **Tip:** Try searching with different parameters or broader categories.")
        
        return "\n".join(output)

# Example usage and setup
def setup_mcp_tools():
    """Setup function to register MCP tools."""
    
    server = AITTestCasesMCPServer()
    
    # Define the MCP tool schema
    tool_schema = {
        "name": server.tool.name,
        "description": server.tool.description,
        "inputSchema": {
            "type": "object",
            "properties": {
                "ait": {
                    "type": "string",
                    "description": "AIT identifier code"
                },
                "ait_name": {
                    "type": "string",
                    "description": "Name of the AIT application"
                },
                "category": {
                    "type": "string",
                    "description": "Testing category (web, mobile, api, etc.)"
                }
            }
        }
    }
    
    return {
        "tools": [tool_schema],
        "handler": server.handle_request
    }

# Environment setup and initialization
def initialize_database_example():
    """Example function to initialize database with sample data."""
    
    config = MongoDBConfig()
    db = config.get_database()
    
    # Sample app_summary data
    app_summary_data = [
        {
            "ait": "AIT-123",
            "ait_name": "Customer Portal",
            "category": "web",
            "description": "Customer-facing web portal",
            "tech_stack": ["React", "Node.js", "MongoDB"],
            "dependencies": ["Auth Service", "Payment Gateway"]
        },
        {
            "ait": "AIT-456",
            "ait_name": "Mobile Banking App",
            "category": "mobile",
            "description": "iOS/Android banking application",
            "tech_stack": ["React Native", "Firebase"],
            "dependencies": ["Core Banking", "Push Notifications"]
        }
    ]
    
    # Sample testing_suggestions data
    testing_suggestions_data = [
        {
            "category": "web",
            "test_type": "Functional Testing",
            "suggestion": "Test all user workflows including login, navigation, and form submissions",
            "priority": "High",
            "complexity": "Medium",
            "estimated_time": "2-3 weeks",
            "tools_recommended": ["Selenium", "Cypress", "Playwright"],
            "test_scenarios": [
                "User registration and login",
                "Form validation and submission",
                "Cross-browser compatibility",
                "Responsive design testing"
            ]
        },
        {
            "category": "mobile",
            "test_type": "Mobile Testing",
            "suggestion": "Test on multiple devices and OS versions",
            "priority": "High",
            "complexity": "High",
            "estimated_time": "3-4 weeks",
            "tools_recommended": ["Appium", "Espresso", "XCUITest"],
            "test_scenarios": [
                "Installation and launch",
                "Touch gestures and responsiveness",
                "Offline functionality",
                "Battery consumption",
                "Memory usage"
            ]
        }
    ]
    
    # Insert sample data (for setup only)
    db['app_summary'].insert_many(app_summary_data)
    db['testing_suggestions'].insert_many(testing_suggestions_data)
    
    print("Database initialized with sample data")

# Main execution for testing
if __name__ == "__main__":
    # Example usage
    tool = AITTestCasesTool()
    
    # Test cases
    test_requests = [
        AITTestCasesRequest(ait="AIT-123"),
        AITTestCasesRequest(category="web"),
        AITTestCasesRequest(ait_name="Mobile Banking"),
        AITTestCasesRequest(ait="AIT-999")  # Non-existent
    ]
    
    for req in test_requests:
        print(f"\n{'='*60}")
        print(f"Testing with: {req.dict(exclude_none=True)}")
        print(f"{'='*60}")
        
        response = tool.process_request(req)
        
        if response.success:
            print(f"✓ {response.message}")
            if response.app_summary:
                print(f"App: {response.app_summary.ait_name}")
            if response.testing_suggestions:
                print(f"Suggestions: {len(response.testing_suggestions)} found")
        else:
            print(f"✗ {response.message}")
```

This implementation provides:

## Key Features:

1. **Complete MCP Tool Structure**:
   - Proper tool description for AI agent selection
   - Clear input parameters with documentation
   - Structured request/response models

2. **Intelligent Routing Logic**:
   - Direct category search when category provided
   - AIT/name lookup to determine category first
   - Case-insensitive regex matching

3. **MongoDB Integration**:
   - Repository pattern for database operations
   - Configurable connection settings
   - Regex-based search on categories

4. **Comprehensive Response Formatting**:
   - Rich text formatting for chat interfaces
   - Detailed application summaries
   - Structured testing suggestions
   - Clear error messages and guidance

5. **Production-Ready Features**:
   - Error handling and validation
   - Type hints and documentation
   - Configurable via environment variables
   - Sample data initialization

## Installation and Setup:

```bash
# Install dependencies
pip install pymongo pydantic

# Set environment variables
export MONGODB_URI="mongodb://localhost:27017"
export MONGODB_DATABASE="ait_testing_db"

# Initialize sample data
python ait_test_cases.py
```

## Usage Examples:

The tool will be automatically selected when users ask about:
- "AIT test cases for our web application"
- "Testing suggestions for mobile apps"
- "What test cases should I write for AIT-123?"
- "Get testing recommendations for API services"

The tool intelligently handles various query patterns and provides comprehensive testing guidance based on application categories and types.
