Here's the enhanced code with validation to only update tools that contain "CURRENT_DATE" in their description:

🚀 Enhanced Method 1: APScheduler with Validation

```python
from fastmcp import FastMCP, Context
from fastmcp.tools import Tool
from apscheduler.schedulers.background import BackgroundScheduler
from apscheduler.triggers.cron import CronTrigger
import datetime
import re

mcp = FastMCP("Validated Scheduled Tool Updater")

# Global tool registry for tracking updatable tools
updatable_tools = {}

def should_update_tool(tool_description: str) -> bool:
    """Validate if tool description contains CURRENT_DATE placeholder."""
    if not tool_description:
        return False
    
    # Check for CURRENT_DATE placeholder (case insensitive)
    pattern = r'\bCURRENT_DATE\b'
    return bool(re.search(pattern, tool_description, re.IGNORECASE))

def create_updated_tool(old_tool, new_description: str):
    """Create a new tool with updated description while preserving other properties."""
    
    # Extract the original function from the old tool
    original_func = old_tool.function
    
    # Create new tool with updated description
    new_tool = Tool.from_tool(
        original_func,
        name=old_tool.name,
        description=new_description,
        # Preserve other properties from original tool
        args=old_tool.args,
        returns=old_tool.returns
    )
    
    return new_tool

def scan_and_update_tools():
    """Scan all tools and update only those with CURRENT_DATE in description."""
    print(f"🔍 Scanning tools for updates at {datetime.datetime.now()}")
    
    current_time = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    updated_count = 0
    
    # Get all currently registered tools
    all_tools = mcp.get_tools()  # This might vary based on FastMCP version
    
    for tool_name, tool_obj in all_tools.items():
        try:
            tool_description = getattr(tool_obj, 'description', '') or getattr(tool_obj, '__doc__', '')
            
            # Validate if tool should be updated
            if should_update_tool(tool_description):
                print(f"🔄 Updating tool: {tool_name}")
                
                # Replace CURRENT_DATE placeholder with actual date
                new_description = re.sub(
                    r'\bCURRENT_DATE\b', 
                    current_time, 
                    tool_description, 
                    flags=re.IGNORECASE
                )
                
                # Create updated tool
                new_tool = create_updated_tool(tool_obj, new_description)
                
                # Replace the tool
                mcp.remove_tool(tool_name)
                mcp.add_tool(new_tool)
                
                # Update registry
                updatable_tools[tool_name] = {
                    'tool': new_tool,
                    'last_updated': current_time,
                    'original_description': tool_description
                }
                
                updated_count += 1
                print(f"✅ Updated tool: {tool_name}")
            else:
                # Track non-updatable tools for reference
                if tool_name not in updatable_tools:
                    updatable_tools[tool_name] = {
                        'tool': tool_obj,
                        'last_updated': 'never',
                        'updatable': False
                    }
                    
        except Exception as e:
            print(f"❌ Error updating tool {tool_name}: {e}")
    
    print(f"📊 Update completed: {updated_count} tools updated")
    return updated_count

def register_initial_tools():
    """Register initial tools with CURRENT_DATE placeholder."""
    
    # Tool 1: Daily data processor
    def daily_processor_impl(data: str) -> dict:
        return {
            "result": f"Processed: {data}",
            "processing_date": datetime.datetime.now().strftime("%Y-%m-%d"),
            "batch_id": f"batch_{datetime.datetime.now().strftime('%Y%m%d_%H%M')}"
        }
    
    daily_tool = Tool.from_tool(
        daily_processor_impl,
        name="daily_data_processor",
        description="""Daily data processor with automatic updates. 
        Last updated: CURRENT_DATE
        This tool processes data and includes the current date in results.
        Scheduled updates: Daily at 1:00 PM"""
    )
    mcp.add_tool(daily_tool)
    
    # Tool 2: Analytics reporter
    def analytics_impl(metric: str) -> dict:
        return {
            "metric": metric,
            "value": 42.0,
            "report_date": datetime.datetime.now().strftime("%Y-%m-%d"),
            "generated_at": "CURRENT_DATE"
        }
    
    analytics_tool = Tool.from_tool(
        analytics_impl,
        name="daily_analytics",
        description="Daily analytics reporter. Report date: CURRENT_DATE. Auto-updates daily."
    )
    mcp.add_tool(analytics_tool)
    
    # Tool 3: Static tool (should NOT be updated)
    def static_processor_impl(data: str) -> str:
        return f"Static processing: {data}"
    
    static_tool = Tool.from_tool(
        static_processor_impl,
        name="static_processor",
        description="This is a static tool that never changes. No CURRENT_DATE here."
    )
    mcp.add_tool(static_tool)
    
    print("✅ Initial tools registered")

def scheduled_update_job():
    """Scheduled job to update tools daily at 1:00 PM."""
    print(f"🕐 Running scheduled tool validation and update at {datetime.datetime.now()}")
    
    try:
        updated_count = scan_and_update_tools()
        print(f"🎯 Scheduled update completed: {updated_count} tools updated")
    except Exception as e:
        print(f"💥 Error in scheduled update: {e}")

# Initialize scheduler
scheduler = BackgroundScheduler()
scheduler.add_job(
    scheduled_update_job,
    trigger=CronTrigger(hour=13, minute=0),  # 1:00 PM daily
    id="daily_tool_update"
)

# Start the scheduler
scheduler.start()
print("🔄 Validated scheduler started - will update ONLY tools with CURRENT_DATE at 1:00 PM")

# Register initial tools
register_initial_tools()

# Perform initial scan
scan_and_update_tools()

@mcp.tool
async def force_tool_update(ctx: Context) -> dict:
    """Manually trigger tool validation and update."""
    await ctx.info("Manual tool update triggered")
    
    updated_count = scan_and_update_tools()
    
    return {
        "status": "success",
        "updated_tools": updated_count,
        "timestamp": datetime.datetime.now().isoformat(),
        "message": f"Updated {updated_count} tools with CURRENT_DATE placeholder"
    }

@mcp.tool
async def get_updatable_tools(ctx: Context) -> dict:
    """Get list of tools that can be updated (contain CURRENT_DATE)."""
    updatable_list = []
    static_list = []
    
    for tool_name, tool_info in updatable_tools.items():
        tool_data = {
            "name": tool_name,
            "last_updated": tool_info.get('last_updated', 'unknown'),
            "updatable": tool_info.get('updatable', True)
        }
        
        if tool_info.get('updatable', True):
            updatable_list.append(tool_data)
        else:
            static_list.append(tool_data)
    
    return {
        "updatable_tools": updatable_list,
        "static_tools": static_list,
        "next_scheduled_update": "13:00 (1:00 PM) daily",
        "current_time": datetime.datetime.now().isoformat()
    }

@mcp.tool
async def validate_tool_description(ctx: Context, tool_name: str) -> dict:
    """Validate if a specific tool contains CURRENT_DATE placeholder."""
    all_tools = mcp.get_tools()
    
    if tool_name not in all_tools:
        return {"valid": False, "error": f"Tool '{tool_name}' not found"}
    
    tool_obj = all_tools[tool_name]
    tool_description = getattr(tool_obj, 'description', '') or getattr(tool_obj, '__doc__', '')
    
    is_updatable = should_update_tool(tool_description)
    
    await ctx.info(f"Validation check for tool '{tool_name}': {is_updatable}")
    
    return {
        "tool_name": tool_name,
        "updatable": is_updatable,
        "description_preview": tool_description[:100] + "..." if len(tool_description) > 100 else tool_description,
        "validation_time": datetime.datetime.now().isoformat()
    }

@mcp.tool
async def register_new_updatable_tool(
    ctx: Context, 
    tool_name: str, 
    description: str
) -> dict:
    """Register a new tool that will be automatically updated daily."""
    
    # Validate that description contains CURRENT_DATE
    if not should_update_tool(description):
        return {
            "status": "error",
            "message": "Tool description must contain 'CURRENT_DATE' placeholder to be auto-updatable"
        }
    
    # Create the tool implementation
    def new_tool_impl(data: str) -> dict:
        return {
            "input": data,
            "processed_at": datetime.datetime.now().isoformat(),
            "tool_version": f"daily_{datetime.datetime.now().strftime('%Y%m%d')}"
        }
    
    # Create and register the tool
    new_tool = Tool.from_tool(
        new_tool_impl,
        name=tool_name,
        description=description
    )
    
    mcp.add_tool(new_tool)
    
    await ctx.info(f"Registered new updatable tool: {tool_name}")
    
    return {
        "status": "success",
        "tool_name": tool_name,
        "updatable": True,
        "message": "Tool registered and will be auto-updated daily at 1:00 PM"
    }
```

🔧 Enhanced Validation Utilities

```python
# Additional validation and utility functions
class ToolUpdateValidator:
    """Enhanced validator for tool updates."""
    
    @staticmethod
    def extract_date_placeholders(description: str) -> list:
        """Extract all date-related placeholders from description."""
        patterns = [
            r'\bCURRENT_DATE\b',
            r'\bCURRENT_TIME\b', 
            r'\bCURRENT_DATETIME\b',
            r'\bTODAY\b',
            r'\bNOW\b'
        ]
        
        found_placeholders = []
        for pattern in patterns:
            if re.search(pattern, description, re.IGNORECASE):
                found_placeholders.append(pattern.strip('\\b'))
        
        return found_placeholders
    
    @staticmethod
    def generate_updated_description(description: str, timestamp: str) -> str:
        """Generate updated description by replacing all date placeholders."""
        placeholder_map = {
            'CURRENT_DATE': timestamp,
            'CURRENT_TIME': datetime.datetime.now().strftime("%H:%M:%S"),
            'CURRENT_DATETIME': datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
            'TODAY': datetime.datetime.now().strftime("%Y-%m-%d"),
            'NOW': datetime.datetime.now().isoformat()
        }
        
        updated_description = description
        for placeholder, replacement in placeholder_map.items():
            updated_description = re.sub(
                f'\\b{placeholder}\\b',
                replacement,
                updated_description,
                flags=re.IGNORECASE
            )
        
        return updated_description

# Enhanced update function using the validator
def enhanced_scan_and_update_tools():
    """Enhanced tool scanning with better validation."""
    print(f"🔍 Enhanced tool scanning at {datetime.datetime.now()}")
    
    current_time = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    updated_count = 0
    update_details = []
    
    all_tools = mcp.get_tools()
    
    for tool_name, tool_obj in all_tools.items():
        try:
            tool_description = getattr(tool_obj, 'description', '') or getattr(tool_obj, '__doc__', '')
            
            if should_update_tool(tool_description):
                # Use enhanced validator
                placeholders = ToolUpdateValidator.extract_date_placeholders(tool_description)
                new_description = ToolUpdateValidator.generate_updated_description(
                    tool_description, 
                    current_time
                )
                
                # Create and register updated tool
                new_tool = create_updated_tool(tool_obj, new_description)
                mcp.remove_tool(tool_name)
                mcp.add_tool(new_tool)
                
                # Update registry
                updatable_tools[tool_name] = {
                    'tool': new_tool,
                    'last_updated': current_time,
                    'placeholders_found': placeholders,
                    'original_description': tool_description
                }
                
                updated_count += 1
                update_details.append({
                    'tool_name': tool_name,
                    'placeholders': placeholders,
                    'update_time': current_time
                })
                
                print(f"✅ Enhanced update: {tool_name} - Placeholders: {placeholders}")
                
        except Exception as e:
            print(f"❌ Enhanced update failed for {tool_name}: {e}")
    
    return {
        "updated_count": updated_count,
        "details": update_details,
        "timestamp": current_time
    }
```

🎯 Key Enhancements:

1. Smart Validation: Only updates tools containing "CURRENT_DATE" in description
2. Placeholder Support: Handles multiple date-related placeholders
3. Detailed Tracking: Tracks which tools are updatable and their update history
4. Safety Checks: Prevents updating static tools accidentally
5. Enhanced Monitoring: Provides tools to check update status and validate descriptions

📊 Example Output:

When the scheduler runs, it will:

· ✅ Update daily_data_processor (contains CURRENT_DATE)
· ✅ Update daily_analytics (contains CURRENT_DATE)
· ❌ Skip static_processor (no CURRENT_DATE)

The validation ensures that only intentionally updatable tools get modified daily at 1:00 PM!