Here are several ways to add a job scheduler to update tools daily at 1:00 PM:

🚀 Method 1: APScheduler (Recommended)

```python
from fastmcp import FastMCP, Context
from fastmcp.tools import Tool
from apscheduler.schedulers.background import BackgroundScheduler
from apscheduler.triggers.cron import CronTrigger
import datetime
import threading

mcp = FastMCP("Scheduled Tool Updater")

# Global tool registry
current_tools = {}

def create_daily_tool():
    """Create a tool with current date in description."""
    current_date = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    
    def daily_processor_impl(data: str) -> dict:
        return {
            "result": f"Processed: {data}",
            "processing_date": current_date,
            "version": "daily_updated"
        }
    
    new_tool = Tool.from_tool(
        daily_processor_impl,
        name="daily_processor",
        description=f"Daily updated data processor. Last refresh: {current_date}"
    )
    return new_tool

def update_tools_job():
    """Scheduled job to update tools daily at 1:00 PM."""
    print(f"🕐 Running scheduled tool update at {datetime.datetime.now()}")
    
    # Remove old tool if exists
    if "daily_processor" in current_tools:
        mcp.remove_tool("daily_processor")
    
    # Create and register new tool with updated date
    new_tool = create_daily_tool()
    mcp.add_tool(new_tool)
    current_tools["daily_processor"] = new_tool
    
    print(f"✅ Tools updated at {datetime.datetime.now()}")

# Initialize scheduler
scheduler = BackgroundScheduler()
scheduler.add_job(
    update_tools_job,
    trigger=CronTrigger(hour=13, minute=0),  # 1:00 PM daily
    id="daily_tool_update"
)

# Start the scheduler
scheduler.start()
print("🔄 Scheduler started - tools will update daily at 1:00 PM")

# Initial tool registration
update_tools_job()

@mcp.tool
async def get_schedule_info(ctx: Context) -> dict:
    """Get information about the scheduled tool updates."""
    job = scheduler.get_job("daily_tool_update")
    next_run = job.next_run_time if job else None
    
    return {
        "scheduled_time": "13:00 (1:00 PM) daily",
        "next_scheduled_run": next_run.isoformat() if next_run else None,
        "last_updated": datetime.datetime.now().isoformat(),
        "active_tools": list(current_tools.keys())
    }
```

🔄 Method 2: Async Scheduler with asyncio

```python
from fastmcp import FastMCP, Context
from fastmcp.tools import Tool
import asyncio
import datetime
import threading

mcp = FastMCP("Async Scheduled Tools")

class AsyncToolScheduler:
    def __init__(self):
        self.current_tools = {}
        self.scheduler_task = None
        self.running = False
    
    async def start_scheduler(self):
        """Start the async scheduler."""
        self.running = True
        self.scheduler_task = asyncio.create_task(self._scheduler_loop())
    
    async def stop_scheduler(self):
        """Stop the async scheduler."""
        self.running = False
        if self.scheduler_task:
            self.scheduler_task.cancel()
    
    async def _scheduler_loop(self):
        """Main scheduler loop that runs daily at 1:00 PM."""
        while self.running:
            now = datetime.datetime.now()
            target_time = now.replace(hour=13, minute=0, second=0, microsecond=0)
            
            # If it's past 1:00 PM, schedule for tomorrow
            if now > target_time:
                target_time += datetime.timedelta(days=1)
            
            wait_seconds = (target_time - now).total_seconds()
            print(f"⏰ Next tool update scheduled for: {target_time}")
            
            # Wait until 1:00 PM
            await asyncio.sleep(wait_seconds)
            
            # Update tools
            await self.update_all_tools()
    
    async def update_all_tools(self):
        """Update all tools with current date."""
        current_time = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        print(f"🔄 Scheduled tool update at {current_time}")
        
        # Update daily processor
        await self.update_daily_processor()
        
        # Add more tool updates here as needed
        await self.update_analytics_tool()
    
    async def update_daily_processor(self):
        """Update the daily processor tool."""
        current_time = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        
        def processor_impl(data: str, mode: str = "standard") -> dict:
            return {
                "input": data,
                "mode": mode,
                "processed_at": current_time,
                "daily_batch": f"batch_{datetime.datetime.now().strftime('%Y%m%d')}"
            }
        
        new_tool = Tool.from_tool(
            processor_impl,
            name="daily_processor",
            description=f"Daily scheduled processor. Last update: {current_time}. Updates daily at 1:00 PM."
        )
        
        # Replace existing tool
        if "daily_processor" in self.current_tools:
            mcp.remove_tool("daily_processor")
        
        mcp.add_tool(new_tool)
        self.current_tools["daily_processor"] = new_tool

    async def update_analytics_tool(self):
        """Update analytics tool with new daily metrics."""
        current_date = datetime.datetime.now().strftime("%Y-%m-%d")
        
        def analytics_impl(metric: str) -> dict:
            daily_data = {
                "user_activity": 1500,
                "processing_volume": 25000,
                "success_rate": 0.987
            }
            return {
                "metric": metric,
                "value": daily_data.get(metric, 0),
                "report_date": current_date,
                "generated_at": datetime.datetime.now().isoformat()
            }
        
        new_tool = Tool.from_tool(
            analytics_impl,
            name="daily_analytics",
            description=f"Daily analytics report. Date: {current_date}. Auto-updates at 1:00 PM."
        )
        
        if "daily_analytics" in self.current_tools:
            mcp.remove_tool("daily_analytics")
        
        mcp.add_tool(new_tool)
        self.current_tools["daily_analytics"] = new_tool

# Initialize scheduler
tool_scheduler = AsyncToolScheduler()

@mcp.tool
async def start_daily_updates(ctx: Context) -> str:
    """Start the daily tool update scheduler."""
    await tool_scheduler.start_scheduler()
    await ctx.info("Daily tool scheduler started - updates at 1:00 PM")
    return "✅ Daily tool updates scheduled for 1:00 PM"

@mcp.tool  
async def stop_daily_updates(ctx: Context) -> str:
    """Stop the daily tool update scheduler."""
    await tool_scheduler.stop_scheduler()
    await ctx.info("Daily tool scheduler stopped")
    return "❌ Daily tool updates stopped"

@mcp.tool
async def manual_update(ctx: Context) -> str:
    """Manually trigger tool updates immediately."""
    await tool_scheduler.update_all_tools()
    await ctx.info("Manual tool update executed")
    return "✅ Tools updated manually"
```

⏰ Method 3: Thread-Based Scheduler

```python
from fastmcp import FastMCP, Context
from fastmcp.tools import Tool
import datetime
import threading
import time

mcp = FastMCP("Thread-Scheduled Tools")

class ThreadedScheduler:
    def __init__(self):
        self.scheduler_thread = None
        self.running = False
        self.tools = {}
    
    def start(self):
        """Start the scheduler in a separate thread."""
        if self.scheduler_thread and self.scheduler_thread.is_alive():
            return
        
        self.running = True
        self.scheduler_thread = threading.Thread(target=self._scheduler_loop, daemon=True)
        self.scheduler_thread.start()
        print("🧵 Thread scheduler started")
    
    def stop(self):
        """Stop the scheduler."""
        self.running = False
    
    def _scheduler_loop(self):
        """Scheduler loop running in background thread."""
        while self.running:
            now = datetime.datetime.now()
            
            # Check if it's 1:00 PM
            if now.hour == 13 and now.minute == 0:
                self._update_tools()
                # Sleep for 61 seconds to avoid multiple executions in the same minute
                time.sleep(61)
            else:
                # Sleep for 30 seconds and check again
                time.sleep(30)
    
    def _update_tools(self):
        """Update tools with current date (run in scheduler thread)."""
        current_time = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        print(f"🔄 Threaded update at {current_time}")
        
        # This would need thread-safe tool operations
        # In practice, you might use a queue to communicate with the main thread
        
        # For demonstration - this shows the concept
        self._create_date_aware_tool(current_time)
    
    def _create_date_aware_tool(self, timestamp: str):
        """Create a tool that's aware of the update schedule."""
        
        def scheduled_processor_impl(query: str) -> dict:
            today = datetime.datetime.now().strftime("%Y-%m-%d")
            return {
                "query": query,
                "result": f"Processed on {today}",
                "scheduled_update": "1:00 PM daily",
                "last_updated": timestamp,
                "next_update": "13:00 tomorrow"
            }
        
        # Note: In production, you'd need thread-safe operations
        # This is simplified for demonstration
        new_tool = Tool.from_tool(
            scheduled_processor_impl,
            name="scheduled_processor",
            description=f"Auto-updating processor. Last: {timestamp}. Next: 1:00 PM"
        )
        
        # Store for reference
        self.tools["scheduled_processor"] = new_tool

# Initialize scheduler
thread_scheduler = ThreadedScheduler()

@mcp.tool
async def initialize_scheduled_updates(ctx: Context) -> str:
    """Initialize and start the daily scheduled tool updates."""
    thread_scheduler.start()
    
    # Create initial tool
    current_time = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    thread_scheduler._create_date_aware_tool(current_time)
    
    await ctx.info(f"Scheduled updates started at {current_time}")
    return "✅ Daily tool updates scheduled for 1:00 PM (thread-based)"
```

🎯 Method 4: Simple FastMCP-Only Solution

```python
from fastmcp import FastMCP, Context
from fastmcp.tools import Tool
import datetime
import asyncio

mcp = FastMCP("Simple Scheduled Tools")

class SimpleScheduler:
    def __init__(self):
        self.update_task = None
    
    async def start_daily_updates(self):
        """Start daily updates at 1:00 PM."""
        if self.update_task:
            self.update_task.cancel()
        
        self.update_task = asyncio.create_task(self._daily_update_loop())
    
    async def _daily_update_loop(self):
        """Simple loop that updates tools daily at 1:00 PM."""
        while True:
            now = datetime.datetime.now()
            target = now.replace(hour=13, minute=0, second=0, microsecond=0)
            
            if now > target:
                target += datetime.timedelta(days=1)
            
            wait_seconds = (target - now).total_seconds()
            print(f"⏰ Next update in {wait_seconds:.0f} seconds")
            
            await asyncio.sleep(wait_seconds)
            await self._update_tools()
    
    async def _update_tools(self):
        """Update all scheduled tools."""
        current_time = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        
        # Update main processor
        await self._update_main_processor(current_time)
        
        print(f"✅ Tools updated at {current_time}")
    
    async def _update_main_processor(self, timestamp: str):
        """Update the main processor tool."""
        
        def main_processor_impl(input_data: str) -> dict:
            return {
                "input": input_data,
                "processed_at": timestamp,
                "daily_version": f"v{datetime.datetime.now().strftime('%Y%m%d')}",
                "next_update": "13:00 tomorrow"
            }
        
        new_tool = Tool.from_tool(
            main_processor_impl,
            name="main_processor",
            description=f"Main data processor. Updated: {timestamp}. Schedule: Daily 1:00 PM"
        )
        
        # Replace tool
        try:
            mcp.remove_tool("main_processor")
        except:
            pass  # Tool might not exist yet
        
        mcp.add_tool(new_tool)

# Initialize scheduler
simple_scheduler = SimpleScheduler()

@mcp.tool
async def schedule_daily_updates(ctx: Context) -> str:
    """Schedule daily tool updates at 1:00 PM."""
    await simple_scheduler.start_daily_updates()
    await ctx.info("Daily updates scheduled for 1:00 PM")
    return "✅ Daily tool updates scheduled for 1:00 PM"

@mcp.tool
async def force_update(ctx: Context) -> str:
    """Force immediate tool update."""
    current_time = datetime.datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    await simple_scheduler._update_tools()
    await ctx.info(f"Forced update at {current_time}")
    return f"✅ Tools updated at {current_time}"
```

📦 Installation Requirements

For the APScheduler method (recommended), add to your requirements:

```bash
pip install apscheduler
```

🎯 Recommendation

Use Method 1 (APScheduler) because:

· Most reliable and battle-tested
· Precise cron scheduling
· Easy to manage and monitor
· Well-documented

The scheduler will automatically update your tool descriptions daily at 1:00 PM with the current timestamp, making it clear when each tool was last updated.

Would you like me to modify any of these approaches for your specific use case?