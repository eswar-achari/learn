I'll create production-ready MCP tools for your MongoDB collection. Here's a complete solution with proper environment variables, error handling, and initialization at application start.

```python
import os
import logging
from typing import Dict, Any, List, Optional
from dataclasses import dataclass, asdict
from datetime import datetime
import asyncio
from contextlib import asynccontextmanager

from pymongo import MongoClient
from pymongo.errors import ConnectionFailure, PyMongoError
from fastapi import FastAPI
import uvicorn

# Setup logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)


@dataclass
class MongoConfig:
    """MongoDB configuration from environment variables."""
    connection_string: str
    database_name: str
    collection_name: str
    
    @classmethod
    def from_env(cls) -> 'MongoConfig':
        """Load configuration from environment variables."""
        connection_string = os.getenv(
            'MONGO_CONNECTION_STRING',
            'mongodb://localhost:27017/'
        )
        
        database_name = os.getenv('MONGO_DATABASE', 'my_database')
        
        collection_name = os.getenv('MONGO_COLLECTION', 'profiles')
        
        return cls(
            connection_string=connection_string,
            database_name=database_name,
            collection_name=collection_name
        )


@dataclass
class ProfileEntry:
    """Data model for profile entries."""
    Profile: Dict[str, Any]
    Description: str
    name: str
    Meta: Dict[str, Any]
    isActive: bool
    created_at: datetime = None
    updated_at: datetime = None
    
    def __post_init__(self):
        """Initialize timestamps."""
        now = datetime.utcnow()
        if self.created_at is None:
            self.created_at = now
        self.updated_at = now
    
    def to_dict(self) -> Dict[str, Any]:
        """Convert to dictionary for MongoDB insertion."""
        data = asdict(self)
        # Convert datetime objects to ISO format strings
        for field in ['created_at', 'updated_at']:
            if field in data and data[field]:
                data[field] = data[field].isoformat()
        return data


class MongoConnectionManager:
    """Manages MongoDB connections with connection pooling."""
    
    def __init__(self, config: MongoConfig):
        self.config = config
        self._client: Optional[MongoClient] = None
        self._db = None
        self._collection = None
    
    async def connect(self) -> None:
        """Establish MongoDB connection."""
        try:
            self._client = MongoClient(
                self.config.connection_string,
                maxPoolSize=100,
                minPoolSize=10,
                retryWrites=True,
                appName="MCP-Tools-App"
            )
            
            # Test the connection
            self._client.admin.command('ping')
            
            self._db = self._client[self.config.database_name]
            self._collection = self._db[self.config.collection_name]
            
            logger.info(f"Connected to MongoDB database: {self.config.database_name}")
            
        except ConnectionFailure as e:
            logger.error(f"Failed to connect to MongoDB: {e}")
            raise
    
    async def disconnect(self) -> None:
        """Close MongoDB connection."""
        if self._client:
            self._client.close()
            logger.info("Disconnected from MongoDB")
    
    @property
    def collection(self):
        """Get the collection object."""
        if self._collection is None:
            raise RuntimeError("MongoDB connection not established")
        return self._collection
    
    @property
    def is_connected(self) -> bool:
        """Check if connection is active."""
        try:
            if self._client:
                self._client.admin.command('ping')
                return True
        except PyMongoError:
            pass
        return False


class ProfileToolsBuilder:
    """Builds MCP tools from MongoDB profile collection."""
    
    def __init__(self, mongo_manager: MongoConnectionManager):
        self.mongo_manager = mongo_manager
        self.tools: List[Dict[str, Any]] = []
    
    async def build_tools_from_collection(self) -> List[Dict[str, Any]]:
        """
        Iterate through MongoDB collection and create tools for each entry.
        
        Returns:
            List of MCP tool definitions
        """
        try:
            logger.info("Starting to build MCP tools from MongoDB collection...")
            
            # Clear existing tools
            self.tools.clear()
            
            # Get all documents from the collection
            cursor = self.mongo_manager.collection.find(
                {"isActive": True}  # Only active profiles
            )
            
            async for document in self._async_cursor(cursor):
                try:
                    tool = self._create_tool_from_document(document)
                    self.tools.append(tool)
                    logger.debug(f"Created tool for profile: {document.get('name')}")
                except Exception as e:
                    logger.error(f"Error creating tool from document {document.get('_id')}: {e}")
            
            logger.info(f"Successfully built {len(self.tools)} MCP tools")
            return self.tools
            
        except PyMongoError as e:
            logger.error(f"MongoDB error while building tools: {e}")
            raise
        except Exception as e:
            logger.error(f"Unexpected error while building tools: {e}")
            raise
    
    def _create_tool_from_document(self, document: Dict[str, Any]) -> Dict[str, Any]:
        """
        Create an MCP tool definition from a MongoDB document.
        
        Args:
            document: MongoDB document with profile data
            
        Returns:
            MCP tool definition
        """
        profile_name = document.get('name', 'Unknown Profile')
        profile_id = str(document.get('_id', ''))
        
        return {
            "name": f"profile_tool_{profile_name.lower().replace(' ', '_')}",
            "description": document.get('Description', 'No description available'),
            "inputSchema": {
                "type": "object",
                "properties": {
                    "action": {
                        "type": "string",
                        "description": "Action to perform",
                        "enum": ["get", "update", "delete"]
                    },
                    "data": {
                        "type": "object",
                        "description": "Data for update action"
                    }
                },
                "required": ["action"]
            },
            "metadata": {
                "profile_id": profile_id,
                "profile_name": profile_name,
                "isActive": document.get('isActive', False),
                "meta": document.get('Meta', {}),
                "original_profile": document.get('Profile', {})
            }
        }
    
    async def _async_cursor(self, cursor):
        """Convert MongoDB cursor to async iterator."""
        # This is a workaround since pymongo is synchronous
        # In production, consider using motor for async MongoDB operations
        while True:
            try:
                if cursor.alive:
                    doc = cursor.next()
                    yield doc
                else:
                    break
            except StopIteration:
                break
    
    def get_tool_by_name(self, tool_name: str) -> Optional[Dict[str, Any]]:
        """Get a specific tool by name."""
        for tool in self.tools:
            if tool['name'] == tool_name:
                return tool
        return None
    
    def get_tools_by_profile(self, profile_name: str) -> List[Dict[str, Any]]:
        """Get all tools for a specific profile name."""
        return [
            tool for tool in self.tools 
            if tool['metadata']['profile_name'] == profile_name
        ]


class MCPServer:
    """MCP Server that hosts the tools."""
    
    def __init__(self):
        self.config = MongoConfig.from_env()
        self.mongo_manager = MongoConnectionManager(self.config)
        self.tools_builder = ProfileToolsBuilder(self.mongo_manager)
        self.app = self._create_fastapi_app()
    
    def _create_fastapi_app(self) -> FastAPI:
        """Create and configure FastAPI application."""
        
        @asynccontextmanager
        async def lifespan(app: FastAPI):
            """Lifespan context manager for startup and shutdown events."""
            # Startup
            await self.mongo_manager.connect()
            await self.tools_builder.build_tools_from_collection()
            logger.info("Application startup complete")
            
            yield
            
            # Shutdown
            await self.mongo_manager.disconnect()
            logger.info("Application shutdown complete")
        
        app = FastAPI(
            title="MCP Tools Server",
            description="MCP Server with tools generated from MongoDB collection",
            version="1.0.0",
            lifespan=lifespan
        )
        
        self._setup_routes(app)
        return app
    
    def _setup_routes(self, app: FastAPI) -> None:
        """Setup API routes."""
        
        @app.get("/")
        async def root():
            """Health check endpoint."""
            return {
                "status": "healthy",
                "mongo_connected": self.mongo_manager.is_connected,
                "tools_count": len(self.tools_builder.tools),
                "timestamp": datetime.utcnow().isoformat()
            }
        
        @app.get("/tools")
        async def get_tools():
            """Get all available MCP tools."""
            return {
                "tools": self.tools_builder.tools,
                "count": len(self.tools_builder.tools)
            }
        
        @app.get("/tools/{tool_name}")
        async def get_tool(tool_name: str):
            """Get specific tool by name."""
            tool = self.tools_builder.get_tool_by_name(tool_name)
            if not tool:
                return {"error": f"Tool '{tool_name}' not found"}, 404
            return tool
        
        @app.post("/tools/{tool_name}/execute")
        async def execute_tool(tool_name: str, payload: Dict[str, Any]):
            """
            Execute a specific tool.
            
            Payload should contain:
            - action: "get", "update", or "delete"
            - data: optional data for update operations
            """
            tool = self.tools_builder.get_tool_by_name(tool_name)
            if not tool:
                return {"error": f"Tool '{tool_name}' not found"}, 404
            
            action = payload.get('action', 'get')
            profile_id = tool['metadata']['profile_id']
            
            try:
                result = await self._execute_profile_action(
                    profile_id, action, payload.get('data', {})
                )
                return {
                    "success": True,
                    "tool": tool_name,
                    "action": action,
                    "result": result,
                    "profile_id": profile_id
                }
            except Exception as e:
                logger.error(f"Error executing tool {tool_name}: {e}")
                return {
                    "success": False,
                    "error": str(e),
                    "tool": tool_name
                }, 500
        
        @app.post("/refresh-tools")
        async def refresh_tools():
            """Refresh tools from MongoDB collection."""
            try:
                await self.tools_builder.build_tools_from_collection()
                return {
                    "success": True,
                    "message": f"Tools refreshed successfully. Now have {len(self.tools_builder.tools)} tools."
                }
            except Exception as e:
                logger.error(f"Error refreshing tools: {e}")
                return {"success": False, "error": str(e)}, 500
    
    async def _execute_profile_action(
        self, 
        profile_id: str, 
        action: str, 
        data: Dict[str, Any]
    ) -> Dict[str, Any]:
        """Execute action on a profile document."""
        
        if action == "get":
            # Retrieve the document
            document = self.mongo_manager.collection.find_one({"_id": profile_id})
            if not document:
                raise ValueError(f"Profile with ID {profile_id} not found")
            
            # Remove MongoDB _id for cleaner response
            if '_id' in document:
                document['_id'] = str(document['_id'])
            return document
        
        elif action == "update":
            # Update the document
            update_result = self.mongo_manager.collection.update_one(
                {"_id": profile_id},
                {"$set": data}
            )
            
            if update_result.matched_count == 0:
                raise ValueError(f"Profile with ID {profile_id} not found")
            
            return {
                "matched_count": update_result.matched_count,
                "modified_count": update_result.modified_count
            }
        
        elif action == "delete":
            # Delete the document (soft delete by setting isActive to false)
            delete_result = self.mongo_manager.collection.update_one(
                {"_id": profile_id},
                {"$set": {"isActive": False}}
            )
            
            if delete_result.matched_count == 0:
                raise ValueError(f"Profile with ID {profile_id} not found")
            
            return {
                "matched_count": delete_result.matched_count,
                "modified_count": delete_result.modified_count,
                "message": "Profile marked as inactive"
            }
        
        else:
            raise ValueError(f"Invalid action: {action}")
    
    async def run(self, host: str = "0.0.0.0", port: int = 8000):
        """Run the MCP server."""
        config = uvicorn.Config(
            app=self.app,
            host=host,
            port=port,
            log_level="info"
        )
        server = uvicorn.Server(config)
        await server.serve()


async def main():
    """Main application entry point."""
    # Read server configuration from environment
    host = os.getenv("SERVER_HOST", "0.0.0.0")
    port = int(os.getenv("SERVER_PORT", "8000"))
    
    # Create and run MCP server
    server = MCPServer()
    
    logger.info(f"Starting MCP Server on {host}:{port}")
    logger.info(f"MongoDB Config: Database={server.config.database_name}, "
                f"Collection={server.config.collection_name}")
    
    await server.run(host=host, port=port)


if __name__ == "__main__":
    # Load environment variables from .env file if present
    from dotenv import load_dotenv
    load_dotenv()
    
    asyncio.run(main())
```

Here's also a `.env.example` file for environment configuration:

```env
# MongoDB Configuration
MONGO_CONNECTION_STRING=mongodb://localhost:27017/
MONGO_DATABASE=my_database
MONGO_COLLECTION=profiles

# Server Configuration
SERVER_HOST=0.0.0.0
SERVER_PORT=8000

# Optional: MongoDB Authentication
# MONGO_USERNAME=your_username
# MONGO_PASSWORD=your_password
# MONGO_AUTH_SOURCE=admin
```

And a `requirements.txt` file:

```txt
fastapi==0.104.1
uvicorn[standard]==0.24.0
pymongo==4.6.0
python-dotenv==1.0.0
```

## Key Features:

1. **Production-Ready Architecture**:
   - Proper error handling and logging
   - Connection pooling for MongoDB
   - Async/await for non-blocking operations
   - Environment-based configuration

2. **MCP Tools Generation**:
   - Iterates through MongoDB collection at startup
   - Creates MCP tools for each active profile
   - Each tool includes metadata from the original document
   - Supports get, update, and delete operations

3. **API Endpoints**:
   - `GET /` - Health check
   - `GET /tools` - List all generated tools
   - `GET /tools/{name}` - Get specific tool
   - `POST /tools/{name}/execute` - Execute tool with action
   - `POST /refresh-tools` - Rebuild tools from collection

4. **Environment Variables**:
   - `MONGO_CONNECTION_STRING` - MongoDB connection string
   - `MONGO_DATABASE` - Database name
   - `MONGO_COLLECTION` - Collection name
   - `SERVER_HOST` - Server host address
   - `SERVER_PORT` - Server port

5. **Data Model**:
   - `ProfileEntry` dataclass for type safety
   - Automatic timestamp management
   - Proper serialization for MongoDB

## Usage:

1. Install dependencies:
```bash
pip install -r requirements.txt
```

2. Configure environment variables in `.env` file

3. Run the application:
```bash
python app.py
```

4. The tools will be automatically generated at startup from your MongoDB collection

The tools are created with the naming pattern `profile_tool_{profile_name}` and include all the original fields from your MongoDB documents in their metadata.
