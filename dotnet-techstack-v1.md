Here's the updated `DotNetDependencyExtractor` class that supports both **local file paths** and **URLs** for `.csproj` files:

```python
import os
import logging
import subprocess
import sys
import json
import urllib.request
import tempfile

class DotNetDependencyExtractor:
    def __init__(self, project_path):
        self.project_path = project_path
        self.is_url = project_path.startswith(('http://', 'https://'))
        self.temp_file = None

    def validate_project_file(self):
        if self.is_url:
            try:
                # Download the file temporarily
                self.temp_file = tempfile.NamedTemporaryFile(suffix=".csproj", delete=False)
                with urllib.request.urlopen(self.project_path) as response:
                    self.temp_file.write(response.read())
                self.temp_file.close()
                self.project_path = self.temp_file.name
                return True
            except Exception as e:
                logging.error(f"Failed to download project file from URL: {e}")
                return False
        else:
            self.project_path = os.path.abspath(self.project_path)
            if not os.path.exists(self.project_path):
                logging.error(f"Project file not found: {self.project_path}")
                return False
            if not self.project_path.endswith(".csproj"):
                logging.error("Please provide a valid .csproj file.")
                return False
            return True

    def extract_dependencies(self):
        try:
            command = [
                "dotnet", "list", self.project_path,
                "package", "--include-transitive"
            ]
            logging.info(f"Running command: {' '.join(command)}")
            result = subprocess.run(command, capture_output=True, text=True, check=True)
            return result.stdout
        except subprocess.CalledProcessError as e:
            logging.error("Failed to extract dependencies.")
            logging.error(e.stderr)
            return None
        finally:
            # Clean up temporary file if it exists
            if self.temp_file and os.path.exists(self.temp_file.name):
                os.unlink(self.temp_file.name)

    def parse_dependencies(self, output):
        dependencies = []
        capture = False
        for line in output.splitlines():
            if line.startswith("Project") or line.startswith("Top-level") or "---" in line:
                if "---" in line:
                    capture = True
                continue

            if capture and "|" in line:
                parts = [part.strip() for part in line.split("|") if part.strip()]
                if len(parts) >= 3:
                    dependencies.append({
                        "name": parts[0],
                        "requested_version": parts[1],
                        "resolved_version": parts[2]
                    })
        return dependencies

    def run(self):
        if not self.validate_project_file():
            return []
        output = self.extract_dependencies()
        if output:
            return self.parse_dependencies(output)
        return []


if __name__ == "__main__":
    if len(sys.argv) != 2:
        print("Usage: python extract_dotnet_deps.py <path-to-csproj-or-url>")
        sys.exit(1)

    project_source = sys.argv[1]
    extractor = DotNetDependencyExtractor(project_source)
    deps = extractor.run()

    print(json.dumps(deps, indent=4))
```

### Key Improvements:

1. **URL Support**:
   - Detects if input is a URL (`http://` or `https://`)
   - Downloads the file to a temporary location
   - Uses the temporary file for dependency analysis

2. **Local Path Support**:
   - Maintains all original functionality for local files
   - Converts relative paths to absolute paths

3. **Automatic Cleanup**:
   - Deletes temporary files after processing
   - Uses Python's `tempfile` module for safe temporary file handling

4. **Error Handling**:
   - Better error messages for URL download failures
   - Maintains all existing validation checks

### Usage Examples:

```bash
# Local file
python extract_dotnet_deps.py ./myproject.csproj

# URL
python extract_dotnet_deps.py https://example.com/projects/myproject.csproj
```

The script will automatically detect whether the input is a local file or URL and handle it appropriately.